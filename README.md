(function() {
    var canvas = document.querySelector("#unity-canvas") || document.querySelector("canvas");
    if (!canvas) {
        console.error("Canvas not found!");
    } else {
        var gl = canvas.getContext("webgl2") || canvas.getContext("webgl");
        if (!gl) {
            console.error("WebGL context not found!");
        } else {
            if (!window._origDrawElements) {
                window._origDrawElements = gl.drawElements;
            }

            window._csMod = {
                wallhack: true,
                chams: "red", // "none", "red", "blue"
                minCount: 300,
                maxCount: 6000,
                log: false
            };

            // Hook WebGL DrawElements
            gl.drawElements = function(mode, count, type, offset) {
                if (window._csMod.log) {
                    console.log("Mesh count: " + count);
                }

                // Filter out map walls and only target player/bot model size ranges
                var isCharacterModel = (count >= window._csMod.minCount && count <= window._csMod.maxCount);

                if (window._csMod.wallhack && isCharacterModel) {
                    gl.disable(gl.DEPTH_TEST);

                    // Optional color tint (Chams)
                    if (window._csMod.chams === "red") {
                        gl.colorMask(true, false, false, true); // Force Red Tint
                    } else if (window._csMod.chams === "blue") {
                        gl.colorMask(false, false, true, true); // Force Blue Tint
                    }

                    window._origDrawElements.apply(this, arguments);

                    // Restore original render state
                    gl.colorMask(true, true, true, true);
                    gl.enable(gl.DEPTH_TEST);
                } else {
                    window._origDrawElements.apply(this, arguments);
                }
            };

            // Create floating GUI on screen
            var oldMenu = document.getElementById("cs-mod-menu");
            if (oldMenu) oldMenu.remove();

            var menu = document.createElement("div");
            menu.id = "cs-mod-menu";
            menu.style.position = "fixed";
            menu.style.top = "10px";
            menu.style.right = "10px";
            menu.style.background = "rgba(0, 0, 0, 0.85)";
            menu.style.border = "2px solid #00ff00";
            menu.style.padding = "10px";
            menu.style.color = "#fff";
            menu.style.fontFamily = "monospace";
            menu.style.zIndex = "999999";
            menu.style.borderRadius = "8px";
            menu.style.fontSize = "12px";

            menu.innerHTML = `
                <div style="font-weight:bold; color:#00ff00; margin-bottom:6px;">CS 1.6 WebGL Tool</div>
                <button id="btn-wh" style="width:100%; margin:2px 0; background:#222; color:#0f0; border:1px solid #444; cursor:pointer;">Wallhack: ON</button>
                <button id="btn-chams" style="width:100%; margin:2px 0; background:#222; color:#f55; border:1px solid #444; cursor:pointer;">Tint: RED</button>
                <button id="btn-log" style="width:100%; margin:2px 0; background:#222; color:#aaa; border:1px solid #444; cursor:pointer;">Log Meshes: OFF</button>
            `;
            document.body.appendChild(menu);

            document.getElementById("btn-wh").onclick = function() {
                window._csMod.wallhack = !window._csMod.wallhack;
                this.innerText = "Wallhack: " + (window._csMod.wallhack ? "ON" : "OFF");
                this.style.color = window._csMod.wallhack ? "#0f0" : "#888";
            };

            document.getElementById("btn-chams").onclick = function() {
                if (window._csMod.chams === "red") {
                    window._csMod.chams = "blue";
                    this.innerText = "Tint: BLUE";
                    this.style.color = "#55f";
                } else if (window._csMod.chams === "blue") {
                    window._csMod.chams = "none";
                    this.innerText = "Tint: OFF";
                    this.style.color = "#888";
                } else {
                    window._csMod.chams = "red";
                    this.innerText = "Tint: RED";
                    this.style.color = "#f55";
                }
            };

            document.getElementById("btn-log").onclick = function() {
                window._csMod.log = !window._csMod.log;
                this.innerText = "Log Meshes: " + (window._csMod.log ? "ON" : "OFF");
                this.style.color = window._csMod.log ? "#ff0" : "#aaa";
            };

            console.log("%c[Success] Mod injected! Look at the top-right corner of the game screen.", "color: #00ff00; font-size: 14px;");
        }
    }
})();
