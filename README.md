# 
Improved meshy GLB downloader
Tampermonkey required.


    // ==UserScript==
    // @name         Meshy GLB Decryptor + Roblox Export
    // @namespace    http://tampermonkey.net/
    // @version      3.0
    // @description  Capture Meshy GLB files and also save a Roblox-ready copy (PNG textures, baked UVs, no extensions)
    // @author       Youssef02
    // @match        https://*.meshy.ai/*
    // @run-at       document-starta
    // @grant        none
    // ==/UserScript==

    (function() {
        'use strict';

    // Inject all interception code directly into the page context via a <script> tag.
    // This is required because Tampermonkey runs in an isolated sandbox — window.Worker
    // overrides set there don't affect the page's own window.Worker.
    const script = document.createElement('script');
    script.textContent = `(${pageScript.toString()})();`;
    (document.head || document.documentElement).appendChild(script);
    script.remove();

    function pageScript() {
        if (window.__meshyDecryptorInjected) return;
        window.__meshyDecryptorInjected = true;

        // ─── SETTINGS ─────────────────────────────────────────────────────────
        const AUTO_DOWNLOAD = true;          // save the original GLB as soon as it is captured
        const AUTO_DOWNLOAD_ROBLOX = true;   // also save the Roblox-ready copy (_roblox.glb)
        const ROBLOX = {
            longestSide: 10,     // studs: the longest side of the model is scaled to this. 0 = keep the original size
            maxTexture: 1024,    // textures are downscaled to this many pixels on the longest side
            maxTriangles: 20000, // only a warning, the mesh is not reduced
            center: true,        // move the model so its bounding box is centred on the origin
        };

        console.log(
            '%c🔓 Meshy Decryptor v3.0 — auto-download ' + (AUTO_DOWNLOAD ? 'enabled' : 'disabled') +
            ', Roblox copy ' + (AUTO_DOWNLOAD_ROBLOX ? 'enabled' : 'disabled'),
            'font-size:16px;color:lime;background:black;padding:4px;'
        );

        window.__meshyGLBs = [];
        window.__meshyLastGLB = null;
        const seen = new Set();

        // ─── DEBUG ────────────────────────────────────────────────────────────
        function dbg(tag, msg, data) {
            const s = 'color:cyan;font-weight:bold;';
            data !== undefined
                ? console.log('%c[Meshy:' + tag + ']', s, msg, data)
                : console.log('%c[Meshy:' + tag + ']', s, msg);
        }

        function isGLBBuffer(buf) {
            if (!buf || buf.byteLength < 4) return false;
            return new Uint32Array(buf.slice ? buf.slice(0, 4) : buf)[0] === 0x46546C67;
        }

        function isMeshyEncrypted(buf) {
            if (!buf || buf.byteLength < 8) return false;
            return new TextDecoder().decode(new Uint8Array(buf, 0, 8)).startsWith('MESHY.AI');
        }

        function looksLikeGLBUrl(url) {
            if (!url || typeof url !== 'string') return false;
            if (url.includes('.glb')) return true;
            if (url.includes('misc/cdn-models')) return true;
            return false;
        }

        // fast fingerprint, so the same model captured through two different hooks is only handled once
        function hashBuffer(buf) {
            const words = new Uint32Array(buf, 0, buf.byteLength >> 2);
            let h = 2166136261 >>> 0;
            for (let i = 0; i < words.length; i++) {
                h ^= words[i];
                h = Math.imul(h, 16777619) >>> 0;
            }
            return h.toString(16) + '-' + buf.byteLength;
        }

        // ─── GLB → ROBLOX CONVERTER ───────────────────────────────────────────
        // What it fixes (all of these make the Roblox importer show errors or garbled textures):
        //   - WebP textures (EXT_texture_webp)              -> re-encoded as PNG, max size ROBLOX.maxTexture
        //   - KHR_texture_transform (UV scale / offset)     -> baked into the UVs (Roblox ignores it, which gives blocky textures)
        //   - KHR_mesh_quantization / interleaved buffers   -> plain float32 positions, normals and UVs
        //   - nested node hierarchy with scale / translation -> flattened into one mesh, transforms applied
        // The converter only needs `convertImage(bytes, mime, maxSize) -> Promise<Uint8Array PNG>`,
        // which keeps it testable outside a browser.
        // <<<CONVERTER
        const COMPONENT = {
            5120: { size: 1, get: (dv, o) => dv.getInt8(o),            norm: 127 },
            5121: { size: 1, get: (dv, o) => dv.getUint8(o),           norm: 255 },
            5122: { size: 2, get: (dv, o) => dv.getInt16(o, true),     norm: 32767 },
            5123: { size: 2, get: (dv, o) => dv.getUint16(o, true),    norm: 65535 },
            5125: { size: 4, get: (dv, o) => dv.getUint32(o, true),    norm: 4294967295 },
            5126: { size: 4, get: (dv, o) => dv.getFloat32(o, true),   norm: 1 },
        };
        const TYPE_COUNT = { SCALAR: 1, VEC2: 2, VEC3: 3, VEC4: 4 };
        const IDENTITY = [1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1];

        function parseGLB(buf) {
            const dv = new DataView(buf);
            if (dv.getUint32(0, true) !== 0x46546C67) throw new Error('not a GLB');
            let off = 12, json = null, bin = null;
            while (off + 8 <= buf.byteLength) {
                const len = dv.getUint32(off, true), type = dv.getUint32(off + 4, true);
                const start = off + 8;
                if (type === 0x4E4F534A) json = JSON.parse(new TextDecoder().decode(new Uint8Array(buf, start, len)));
                else if (type === 0x004E4942) bin = new Uint8Array(buf, start, len);
                off = start + len;
            }
            if (!json) throw new Error('GLB has no JSON chunk');
            return { json, bin };
        }

        // Reads any accessor (any component type, normalized or not, interleaved or not).
        // raw = true gives the integer values untouched (used for indices).
        function readAccessor(ctx, index, raw) {
            const acc = ctx.json.accessors[index];
            const comp = COMPONENT[acc.componentType];
            const n = TYPE_COUNT[acc.type];
            if (!comp || !n) throw new Error('unsupported accessor type');
            if (acc.sparse) throw new Error('sparse accessors are not supported');
            const out = raw ? new Uint32Array(acc.count * n) : new Float32Array(acc.count * n);
            if (acc.bufferView === undefined) return out;
            const bv = ctx.json.bufferViews[acc.bufferView];
            const base = ctx.bin.byteOffset + (bv.byteOffset || 0) + (acc.byteOffset || 0);
            const stride = bv.byteStride || comp.size * n;
            const dv = new DataView(ctx.bin.buffer);
            for (let i = 0; i < acc.count; i++) {
                for (let k = 0; k < n; k++) {
                    let v = comp.get(dv, base + i * stride + k * comp.size);
                    if (!raw && acc.normalized) v = Math.max(v / comp.norm, -1);
                    out[i * n + k] = v;
                }
            }
            return out;
        }

        // 4x4 column-major matrices
        function mat4FromTRS(t, r, s) {
            const x = r[0], y = r[1], z = r[2], w = r[3];
            const x2 = x + x, y2 = y + y, z2 = z + z;
            const xx = x * x2, xy = x * y2, xz = x * z2, yy = y * y2, yz = y * z2, zz = z * z2;
            const wx = w * x2, wy = w * y2, wz = w * z2;
            return [
                (1 - (yy + zz)) * s[0], (xy + wz) * s[0], (xz - wy) * s[0], 0,
                (xy - wz) * s[1], (1 - (xx + zz)) * s[1], (yz + wx) * s[1], 0,
                (xz + wy) * s[2], (yz - wx) * s[2], (1 - (xx + yy)) * s[2], 0,
                t[0], t[1], t[2], 1,
            ];
        }

        function mat4Mul(a, b) {
            const o = new Array(16).fill(0);
            for (let c = 0; c < 4; c++) {
                for (let r = 0; r < 4; r++) {
                    let s = 0;
                    for (let k = 0; k < 4; k++) s += a[k * 4 + r] * b[c * 4 + k];
                    o[c * 4 + r] = s;
                }
            }
            return o;
        }

        // inverse-transpose of the upper 3x3 (row-major result) and the determinant
        function normalMatrix(m) {
            const a = m[0], b = m[1], c = m[2], d = m[4], e = m[5], f = m[6], g = m[8], h = m[9], i = m[10];
            const c00 = e * i - h * f, c01 = -(b * i - h * c), c02 = b * f - e * c;
            const c10 = -(d * i - g * f), c11 = a * i - g * c, c12 = -(a * f - d * c);
            const c20 = d * h - g * e, c21 = -(a * h - g * b), c22 = a * e - d * b;
            const det = a * c00 + d * c01 + g * c02;
            const inv = det !== 0 ? 1 / det : 1;
            return { det, n: [c00 * inv, c01 * inv, c02 * inv, c10 * inv, c11 * inv, c12 * inv, c20 * inv, c21 * inv, c22 * inv] };
        }

        // KHR_texture_transform: uv' = T * R * S * uv
        function bakeUV(uv, tt) {
            const off = tt.offset || [0, 0], rot = tt.rotation || 0, sc = tt.scale || [1, 1];
            const cs = Math.cos(rot), sn = Math.sin(rot);
            for (let i = 0; i < uv.length; i += 2) {
                const u = uv[i] * sc[0], v = uv[i + 1] * sc[1];
                uv[i] = cs * u + sn * v + off[0];
                uv[i + 1] = -sn * u + cs * v + off[1];
            }
        }

        function getTransform(info) {
            return info && info.extensions && info.extensions.KHR_texture_transform
                ? info.extensions.KHR_texture_transform : null;
        }

        // A mesh has one set of UVs, so only one transform can be baked. Use the base color one.
        function pickTransform(mat, log) {
            const pbr = mat.pbrMetallicRoughness || {};
            const infos = [pbr.baseColorTexture, pbr.metallicRoughnessTexture, mat.normalTexture, mat.occlusionTexture, mat.emissiveTexture]
                .filter(Boolean);
            const first = infos.map(getTransform).find(Boolean) || null;
            const key = JSON.stringify(first);
            if (infos.some(t => JSON.stringify(getTransform(t)) !== key)) {
                log('warning: the textures use different UV transforms, only one could be baked');
            }
            return first;
        }

        async function cleanGLBForRoblox(arrayBuffer, opts, convertImage) {
            const log = opts.log || function() {};
            const { json, bin } = parseGLB(arrayBuffer);
            const ctx = { json, bin };

            // 1. collect every mesh primitive together with its world matrix
            const prims = [];
            function visit(ni, parent) {
                const node = json.nodes[ni];
                const local = node.matrix ? node.matrix.slice()
                    : mat4FromTRS(node.translation || [0, 0, 0], node.rotation || [0, 0, 0, 1], node.scale || [1, 1, 1]);
                const world = mat4Mul(parent, local);
                if (node.mesh !== undefined) {
                    for (const p of json.meshes[node.mesh].primitives) prims.push({ p, world });
                }
                for (const c of node.children || []) visit(c, world);
            }
            let roots;
            if (json.scenes && json.scenes.length) {
                roots = json.scenes[json.scene || 0].nodes || [];
            } else {
                const children = new Set();
                (json.nodes || []).forEach(n => (n.children || []).forEach(c => children.add(c)));
                roots = (json.nodes || []).map((_, i) => i).filter(i => !children.has(i));
            }
            roots.forEach(r => visit(r, IDENTITY));

            // 2. read the geometry, apply the node transforms, bake the UV transform
            const geo = [];
            for (const { p, world } of prims) {
                if ((p.mode !== undefined && p.mode !== 4) || p.attributes.POSITION === undefined) {
                    log('skipped a primitive that is not a triangle list');
                    continue;
                }
                const pos = readAccessor(ctx, p.attributes.POSITION);
                const count = pos.length / 3;
                const nrm = p.attributes.NORMAL !== undefined ? readAccessor(ctx, p.attributes.NORMAL) : null;
                const uv = p.attributes.TEXCOORD_0 !== undefined ? readAccessor(ctx, p.attributes.TEXCOORD_0) : new Float32Array(count * 2);
                let idx = p.indices !== undefined ? readAccessor(ctx, p.indices, true) : Uint32Array.from({ length: count }, (_, i) => i);

                const m = world, nm = normalMatrix(world);
                for (let i = 0; i < count; i++) {
                    const x = pos[i * 3], y = pos[i * 3 + 1], z = pos[i * 3 + 2];
                    pos[i * 3]     = m[0] * x + m[4] * y + m[8] * z + m[12];
                    pos[i * 3 + 1] = m[1] * x + m[5] * y + m[9] * z + m[13];
                    pos[i * 3 + 2] = m[2] * x + m[6] * y + m[10] * z + m[14];
                    if (nrm) {
                        const nx = nrm[i * 3], ny = nrm[i * 3 + 1], nz = nrm[i * 3 + 2];
                        let ax = nm.n[0] * nx + nm.n[1] * ny + nm.n[2] * nz;
                        let ay = nm.n[3] * nx + nm.n[4] * ny + nm.n[5] * nz;
                        let az = nm.n[6] * nx + nm.n[7] * ny + nm.n[8] * nz;
                        const len = Math.hypot(ax, ay, az) || 1;
                        nrm[i * 3] = ax / len; nrm[i * 3 + 1] = ay / len; nrm[i * 3 + 2] = az / len;
                    }
                }
                if (nm.det < 0) { // mirrored node: flip the winding so the faces do not turn inside out
                    for (let i = 0; i + 2 < idx.length; i += 3) { const t = idx[i + 1]; idx[i + 1] = idx[i + 2]; idx[i + 2] = t; }
                }

                const mat = p.material !== undefined ? json.materials[p.material] : {};
                const tt = pickTransform(mat, log);
                if (tt) bakeUV(uv, tt);
                geo.push({ pos, nrm, uv, idx, material: p.material });
            }
            if (!geo.length) throw new Error('no triangle meshes found in this GLB');

            // 3. scale and centre
            const mn = [Infinity, Infinity, Infinity], mx = [-Infinity, -Infinity, -Infinity];
            for (const g of geo) {
                for (let i = 0; i < g.pos.length; i += 3) {
                    for (let k = 0; k < 3; k++) {
                        mn[k] = Math.min(mn[k], g.pos[i + k]);
                        mx[k] = Math.max(mx[k], g.pos[i + k]);
                    }
                }
            }
            const longest = Math.max(mx[0] - mn[0], mx[1] - mn[1], mx[2] - mn[2]) || 1;
            const k = opts.longestSide > 0 ? opts.longestSide / longest : 1;
            const ctr = opts.center ? [(mn[0] + mx[0]) / 2, (mn[1] + mx[1]) / 2, (mn[2] + mx[2]) / 2] : [0, 0, 0];
            for (const g of geo) {
                for (let i = 0; i < g.pos.length; i += 3) {
                    for (let c = 0; c < 3; c++) g.pos[i + c] = (g.pos[i + c] - ctr[c]) * k;
                }
            }

            // 4. output buffer writer
            const parts = [];
            const bufferViews = [];
            let size = 0;
            function addView(u8, target) {
                const pad = (4 - (size % 4)) % 4;
                if (pad) { parts.push(new Uint8Array(pad)); size += pad; }
                const v = { buffer: 0, byteOffset: size, byteLength: u8.byteLength };
                if (target) v.target = target;
                parts.push(u8);
                size += u8.byteLength;
                bufferViews.push(v);
                return bufferViews.length - 1;
            }
            const bytesOf = ta => new Uint8Array(ta.buffer, ta.byteOffset, ta.byteLength);

            // 5. textures -> PNG
            const images = [], textures = [];
            const imageMap = new Map(), texMap = new Map();

            function getImageBytes(src) {
                const im = json.images[src];
                if (im.bufferView !== undefined) {
                    const bv = json.bufferViews[im.bufferView];
                    const start = bin.byteOffset + (bv.byteOffset || 0);
                    return { data: new Uint8Array(bin.buffer, start, bv.byteLength), mime: im.mimeType || 'image/png' };
                }
                if (im.uri && im.uri.startsWith('data:')) {
                    const m = /^data:([^;,]+)?(;base64)?,(.*)$/.exec(im.uri);
                    const bin2 = atob(decodeURIComponent(m[3]));
                    const data = new Uint8Array(bin2.length);
                    for (let i = 0; i < bin2.length; i++) data[i] = bin2.charCodeAt(i);
                    return { data, mime: m[1] || 'image/png' };
                }
                throw new Error('external image files are not supported');
            }

            async function useTexture(ti) {
                if (ti === undefined) return undefined;
                if (texMap.has(ti)) return texMap.get(ti);
                const t = json.textures[ti];
                let src = t.source;
                if (src === undefined && t.extensions && t.extensions.EXT_texture_webp) src = t.extensions.EXT_texture_webp.source;
                if (src === undefined) throw new Error('texture ' + ti + ' has no image Roblox can read (KTX2/Basis is not supported)');
                let imgIdx = imageMap.get(src);
                if (imgIdx === undefined) {
                    const { data, mime } = getImageBytes(src);
                    log('converting image ' + src + ' (' + mime + ', ' + data.byteLength + ' bytes) to PNG');
                    const png = await convertImage(data, mime, opts.maxTexture);
                    images.push({ bufferView: addView(png), mimeType: 'image/png', name: json.images[src].name || ('image' + src) });
                    imgIdx = images.length - 1;
                    imageMap.set(src, imgIdx);
                }
                textures.push({ source: imgIdx, sampler: 0 });
                texMap.set(ti, textures.length - 1);
                return textures.length - 1;
            }

            const matMap = new Map(), outMats = [];
            async function useMaterial(mi) {
                const key = mi === undefined ? -1 : mi;
                if (matMap.has(key)) return matMap.get(key);
                const m = mi === undefined ? {} : json.materials[mi];
                const pbr = m.pbrMetallicRoughness || {};
                const om = { name: m.name || ('Material' + key), doubleSided: !!m.doubleSided, pbrMetallicRoughness: {} };
                const bc = await useTexture(pbr.baseColorTexture && pbr.baseColorTexture.index);
                if (bc !== undefined) om.pbrMetallicRoughness.baseColorTexture = { index: bc };
                const mr = await useTexture(pbr.metallicRoughnessTexture && pbr.metallicRoughnessTexture.index);
                if (mr !== undefined) om.pbrMetallicRoughness.metallicRoughnessTexture = { index: mr };
                ['baseColorFactor', 'metallicFactor', 'roughnessFactor'].forEach(f => {
                    if (pbr[f] !== undefined) om.pbrMetallicRoughness[f] = pbr[f];
                });
                const nt = await useTexture(m.normalTexture && m.normalTexture.index);
                if (nt !== undefined) {
                    om.normalTexture = { index: nt };
                    if (m.normalTexture.scale !== undefined) om.normalTexture.scale = m.normalTexture.scale;
                }
                const ot = await useTexture(m.occlusionTexture && m.occlusionTexture.index);
                if (ot !== undefined) om.occlusionTexture = { index: ot };
                const et = await useTexture(m.emissiveTexture && m.emissiveTexture.index);
                if (et !== undefined) om.emissiveTexture = { index: et };
                if (m.emissiveFactor) om.emissiveFactor = m.emissiveFactor;
                if (m.alphaMode && m.alphaMode !== 'OPAQUE') {
                    om.alphaMode = m.alphaMode;
                    if (m.alphaCutoff !== undefined) om.alphaCutoff = m.alphaCutoff;
                }
                outMats.push(om);
                matMap.set(key, outMats.length - 1);
                return outMats.length - 1;
            }

            // 6. write the primitives
            const accessors = [];
            const primitives = [];
            let tris = 0, verts = 0;
            for (const g of geo) {
                const count = g.pos.length / 3;
                const pMin = [Infinity, Infinity, Infinity], pMax = [-Infinity, -Infinity, -Infinity];
                for (let i = 0; i < g.pos.length; i += 3) {
                    for (let c = 0; c < 3; c++) { pMin[c] = Math.min(pMin[c], g.pos[i + c]); pMax[c] = Math.max(pMax[c], g.pos[i + c]); }
                }
                const attributes = {};
                accessors.push({ bufferView: addView(bytesOf(g.pos), 34962), componentType: 5126, count, type: 'VEC3', min: pMin, max: pMax });
                attributes.POSITION = accessors.length - 1;
                if (g.nrm) {
                    accessors.push({ bufferView: addView(bytesOf(g.nrm), 34962), componentType: 5126, count, type: 'VEC3' });
                    attributes.NORMAL = accessors.length - 1;
                }
                accessors.push({ bufferView: addView(bytesOf(g.uv), 34962), componentType: 5126, count, type: 'VEC2' });
                attributes.TEXCOORD_0 = accessors.length - 1;
                const wide = count > 65535;
                const idx = wide ? Uint32Array.from(g.idx) : Uint16Array.from(g.idx);
                accessors.push({ bufferView: addView(bytesOf(idx), 34963), componentType: wide ? 5125 : 5123, count: idx.length, type: 'SCALAR' });
                primitives.push({ attributes, indices: accessors.length - 1, material: await useMaterial(g.material) });
                tris += idx.length / 3;
                verts += count;
            }
            if (tris > opts.maxTriangles) log('warning: ' + tris + ' triangles, Roblox meshes should stay under ~' + opts.maxTriangles + '. Reduce them in Blender (Decimate) first');

            // 7. assemble the GLB
            const gltf = {
                asset: { version: '2.0', generator: 'meshy-roblox-export' },
                scene: 0,
                scenes: [{ nodes: [0] }],
                nodes: [{ mesh: 0, name: opts.name || 'Model' }],
                meshes: [{ name: opts.name || 'Model', primitives }],
                materials: outMats,
                accessors,
                bufferViews,
            };
            if (textures.length) {
                gltf.textures = textures;
                gltf.images = images;
                gltf.samplers = [{ magFilter: 9729, minFilter: 9987, wrapS: 10497, wrapT: 10497 }];
            }
            let binLen = size;
            while (binLen % 4) binLen++;
            gltf.buffers = [{ byteLength: binLen }];
            let jsonBytes = new TextEncoder().encode(JSON.stringify(gltf));
            const jsonPad = (4 - (jsonBytes.length % 4)) % 4;
            if (jsonPad) { const padded = new Uint8Array(jsonBytes.length + jsonPad).fill(0x20); padded.set(jsonBytes); jsonBytes = padded; }

            const total = 12 + 8 + jsonBytes.length + 8 + binLen;
            const out = new Uint8Array(total);
            const dv = new DataView(out.buffer);
            dv.setUint32(0, 0x46546C67, true); dv.setUint32(4, 2, true); dv.setUint32(8, total, true);
            dv.setUint32(12, jsonBytes.length, true); dv.setUint32(16, 0x4E4F534A, true);
            out.set(jsonBytes, 20);
            const binHead = 20 + jsonBytes.length;
            dv.setUint32(binHead, binLen, true); dv.setUint32(binHead + 4, 0x004E4942, true);
            let o = binHead + 8, written = 0;
            for (const part of parts) { out.set(part, o); o += part.byteLength; written += part.byteLength; }

            return {
                data: out,
                info: { triangles: tris, vertices: verts, images: images.length, size: [0, 1, 2].map(c => (mx[c] - mn[c]) * k) },
            };
        }
        // CONVERTER>>>

        // Browser image conversion: WebP/JPEG/PNG in, downscaled PNG out.
        // colorSpaceConversion 'none' keeps normal maps and metal/roughness maps byte-exact.
        async function convertImageToPNG(bytes, mime, maxSize) {
            const bmp = await createImageBitmap(new Blob([bytes], { type: mime }), {
                colorSpaceConversion: 'none',
                premultiplyAlpha: 'none',
            });
            const scale = Math.min(1, maxSize / Math.max(bmp.width, bmp.height));
            const w = Math.max(1, Math.round(bmp.width * scale));
            const h = Math.max(1, Math.round(bmp.height * scale));
            let blob;
            if (typeof OffscreenCanvas !== 'undefined') {
                const c = new OffscreenCanvas(w, h);
                const g = c.getContext('2d');
                g.imageSmoothingQuality = 'high';
                g.drawImage(bmp, 0, 0, w, h);
                blob = await c.convertToBlob({ type: 'image/png' });
            } else {
                const c = document.createElement('canvas');
                c.width = w; c.height = h;
                const g = c.getContext('2d');
                g.imageSmoothingQuality = 'high';
                g.drawImage(bmp, 0, 0, w, h);
                blob = await new Promise(res => c.toBlob(res, 'image/png'));
            }
            bmp.close && bmp.close();
            return new Uint8Array(await blob.arrayBuffer());
        }

        // ─── HELPERS ──────────────────────────────────────────────────────────
        function getModelFilename() {
            const title = document.querySelector('h1')?.textContent || '';
            const safeTitle = title
                .replace(/[<>:"/\\|?*\x00-\x1F-]+/g, '-')
                .replace(/\s+/g, ' ')
                .replace(/^[.\s-]+|[.\s-]+$/g, '');

            return safeTitle ? safeTitle + '.glb' : 'meshy_' + Date.now() + '.glb';
        }

        function downloadGLB(blob, filename, revokeDelay = 10000) {
            const url = _origCreateObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = filename;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            setTimeout(() => URL.revokeObjectURL(url), revokeDelay);
        }

        // ─── CAPTURE ──────────────────────────────────────────────────────────
        async function makeRobloxCopy(entry, buf) {
            try {
                const base = entry.filename.replace(/\.glb$/i, '');
                const res = await cleanGLBForRoblox(
                    buf,
                    Object.assign({ name: base.replace(/[^\w-]+/g, '_').slice(0, 40) || 'Model', log: m => dbg('ROBLOX', m) }, ROBLOX),
                    convertImageToPNG
                );
                entry.robloxBlob = new Blob([res.data], { type: 'model/gltf-binary' });
                entry.robloxName = base + '_roblox.glb';
                entry.robloxInfo = res.info;
                dbg('ROBLOX', 'ready: ' + res.info.triangles + ' tris, ' + res.info.images + ' PNG textures, size ' +
                    res.info.size.map(v => v.toFixed(1)).join(' x ') + ', ' + res.data.byteLength + ' bytes');
                if (AUTO_DOWNLOAD_ROBLOX) downloadGLB(entry.robloxBlob, entry.robloxName);
            } catch (ex) {
                entry.robloxError = String(ex && ex.message || ex);
                dbg('ROBLOX', 'conversion failed: ' + entry.robloxError);
            }
            updateButton();
        }

        async function captureGLB(blob, src) {
            let buf = null;
            try { buf = await blob.arrayBuffer(); } catch (e) { dbg('CAPTURE', 'could not read blob: ' + e); }
            if (buf) {
                const sig = hashBuffer(buf);
                if (seen.has(sig)) { dbg('CAPTURE', 'duplicate ignored (' + src + ')'); return; }
                seen.add(sig);
            }
            const entry = { blob, src, time: Date.now(), filename: getModelFilename() };
            window.__meshyGLBs.push(entry);
            window.__meshyLastGLB = entry;
            dbg('CAPTURE', 'GLB captured! filename=' + entry.filename + ' size=' + blob.size + ' src=' + src);
            updateButton();
            if (AUTO_DOWNLOAD) {
                downloadGLB(blob, entry.filename);
            }
            if (buf) makeRobloxCopy(entry, buf);
        }

        // ─── HOOK: Worker ─────────────────────────────────────────────────────
        const _OrigWorker = window.Worker;
        window.Worker = function MeshyWorkerProxy(scriptURL, options) {
            const w = new _OrigWorker(scriptURL, options);
            dbg('WORKER', 'Created: ' + String(scriptURL).slice(-80));

            // Wrap onmessage via defineProperty on the prototype descriptor
            const proto = Object.getPrototypeOf(w);
            const desc = Object.getOwnPropertyDescriptor(proto, 'onmessage');
            if (desc && desc.set) {
                Object.defineProperty(w, 'onmessage', {
                    configurable: true,
                    get() { return desc.get ? desc.get.call(w) : undefined; },
                    set(fn) {
                        desc.set.call(w, function(e) {
                            spyWorkerMsg(e);
                            return fn.call(this, e);
                        });
                    }
                });
            }

            // Also wrap addEventListener
            const _origAEL = w.addEventListener.bind(w);
            w.addEventListener = function(type, listener, opts) {
                if (type === 'message') {
                    return _origAEL(type, function(e) {
                        spyWorkerMsg(e);
                        return listener.call(this, e);
                    }, opts);
                }
                return _origAEL(type, listener, opts);
            };

            return w;
        };
        window.Worker.prototype = _OrigWorker.prototype;
        // Make sure new Worker(...) instanceof Worker still works
        Object.defineProperty(window.Worker, Symbol.hasInstance, {
            value: function(instance) { return instance instanceof _OrigWorker; }
        });

        function spyWorkerMsg(e) {
            const d = e.data;
            if (!d || !d.type) return;
            dbg('WORKER', 'msg type=' + d.type + (d.error ? ' err=' + d.error : '') + (d.success !== undefined ? ' ok=' + d.success : ''));
            if (d.type === 'process' && d.success && d.data) {
                dbg('WORKER', 'DECRYPTED! size=' + d.data.byteLength);
                try {
                    const copy = d.data.slice(0); // copy before app uses it
                    captureGLB(new Blob([copy], { type: 'model/gltf-binary' }), 'worker');
                } catch(ex) { dbg('WORKER', 'capture error: ' + ex); }
            }
        }

        // ─── HOOK: fetch ──────────────────────────────────────────────────────
        const _origFetch = window.fetch;
        window.fetch = async function(input) {
            const url = typeof input === 'string' ? input : (input && input.url) || String(input);
            const hit = looksLikeGLBUrl(url);
            if (hit) dbg('FETCH', 'intercepting: ' + url.slice(-80));
            const p = _origFetch.apply(this, arguments);
            if (!hit) return p;
            return p.then(async resp => {
                try {
                    const buf = await resp.clone().arrayBuffer();
                    dbg('FETCH', 'size=' + buf.byteLength + ' encrypted=' + isMeshyEncrypted(buf) + ' glb=' + isGLBBuffer(buf));
                    if (isGLBBuffer(buf)) {
                        captureGLB(new Blob([buf], { type: 'model/gltf-binary' }), 'fetch');
                    }
                } catch(ex) { dbg('FETCH', 'read err: ' + ex); }
                return resp;
            });
        };

        // ─── HOOK: createObjectURL ────────────────────────────────────────────
        const _origCreateObjectURL = URL.createObjectURL.bind(URL);
        URL.createObjectURL = function(blob) {
            const url = _origCreateObjectURL(blob);
            if (blob instanceof Blob || blob instanceof File) {
                if (blob.size > 1000) {
                    dbg('BLOB', 'type=' + blob.type + ' size=' + blob.size);
                }
                if (blob.type === 'model/gltf-binary') {
                    dbg('BLOB', 'GLB blob! capturing...');
                    captureGLB(blob, 'createObjectURL');
                } else if (blob.size > 100000) {
                    blob.arrayBuffer().then(buf => {
                        if (isGLBBuffer(buf)) {
                            dbg('BLOB', 'GLB magic in untyped blob!');
                            captureGLB(new Blob([buf], { type: 'model/gltf-binary' }), 'createObjectURL-magic');
                        }
                    }).catch(() => {});
                }
            }
            return url;
        };

        // ─── UI ───────────────────────────────────────────────────────────────
        function updateButton() {
            const panel = document.getElementById('meshy-decrypt-panel');
            if (!panel) return;
            const glbs = window.__meshyGLBs;
            const n = glbs.length;
            const last = window.__meshyLastGLB;
            const rb = panel.querySelector('#meshy-roblox-btn');
            const ob = panel.querySelector('#meshy-orig-btn');
            const st = panel.querySelector('#meshy-status');

            const ready = glbs.filter(e => e.robloxBlob).length;
            rb.textContent = n === 0 ? '🎮 Waiting for GLB…' : '🎮 Download Roblox-ready' + (ready > 1 ? ' (' + ready + ')' : '');
            rb.disabled = ready === 0;
            rb.style.background = ready > 0 ? '#00ff00' : '#888';
            rb.style.cursor = ready > 0 ? 'pointer' : 'default';

            ob.textContent = '💾 Original' + (n > 1 ? ' (' + n + ')' : '');
            ob.disabled = n === 0;
            ob.style.background = n > 0 ? '#9be7ff' : '#888';
            ob.style.cursor = n > 0 ? 'pointer' : 'default';

            if (!last) st.textContent = '';
            else if (last.robloxError) st.textContent = 'Roblox copy failed: ' + last.robloxError;
            else if (last.robloxInfo) {
                const i = last.robloxInfo;
                st.textContent = i.triangles.toLocaleString() + ' tris · ' + i.size.map(v => v.toFixed(1)).join('×') + ' studs' +
                    (i.triangles > ROBLOX.maxTriangles ? ' · too many tris!' : '');
            } else st.textContent = 'Converting…';
        }

        function addButton() {
            if (document.getElementById('meshy-decrypt-panel')) return;
            if (!document.querySelector('canvas')) { setTimeout(addButton, 1000); return; }

            const panel = document.createElement('div');
            panel.id = 'meshy-decrypt-panel';
            panel.style.cssText = 'position:fixed;bottom:20px;right:20px;z-index:999999;display:flex;flex-direction:column;gap:6px;align-items:stretch;font-family:sans-serif;';

            const btnCss = 'padding:10px 20px;background:#888;color:#000;border:none;border-radius:8px;font-weight:bold;font-size:14px;';
            const rb = document.createElement('button');
            rb.id = 'meshy-roblox-btn';
            rb.style.cssText = btnCss;
            const ob = document.createElement('button');
            ob.id = 'meshy-orig-btn';
            ob.style.cssText = btnCss + 'font-size:12px;padding:6px 20px;';
            const st = document.createElement('div');
            st.id = 'meshy-status';
            st.style.cssText = 'color:#fff;background:rgba(0,0,0,.65);border-radius:6px;padding:3px 8px;font-size:11px;text-align:center;max-width:260px;';

            function save(kind) {
                const glbs = window.__meshyGLBs;
                if (!glbs.length) {
                    alert('No GLB captured yet.\n\nOpen a model, then try the Export/Download button first.\nCheck F12 console for [Meshy:*] logs.');
                    return;
                }
                const list = glbs.filter(e => kind === 'roblox' ? e.robloxBlob : true);
                list.forEach((entry, i) => {
                    const blob = kind === 'roblox' ? entry.robloxBlob : entry.blob;
                    let name = kind === 'roblox' ? entry.robloxName : entry.filename;
                    if (list.length > 1) name = name.replace(/\.glb$/i, '_' + (i + 1) + '.glb');
                    downloadGLB(blob, name, 5000);
                });
            }
            rb.onclick = () => save('roblox');
            ob.onclick = () => save('original');

            panel.append(rb, ob, st);
            document.body.appendChild(panel);
            dbg('UI', 'Panel added');
            updateButton();
        }

        setTimeout(addButton, 2000);
    }
})();
