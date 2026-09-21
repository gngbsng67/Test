(function() {
    var canvas = document.querySelector("#unity-canvas") || document.querySelector("canvas");
    if (!canvas) {
        console.error("Canvas not found!");
    } else {
        var gl = canvas.getContext("webgl2") || canvas.getContext("webgl");
        if (!gl) {
            console.error("WebGL context not found!");
        } else {
            // Save original rendering functions
            if (!window._origDrawElements) {
                window._origDrawElements = gl.drawElements;
            }

            // --- 1. SPEED HACK ENGINE ---
            if (!window._origPerfNow) {
                window._origPerfNow = performance.now.bind(performance);
                window._gameSpeed = 1.0;
                var lastRealTime = window._origPerfNow();
                var virtualTime = window._origPerfNow();

                performance.now = function() {
                    var currentRealTime = window._origPerfNow();
                    var delta = currentRealTime - lastRealTime;
                    lastRealTime = currentRealTime;
                    virtualTime += delta * window._gameSpeed;
                    return virtualTime;
                };
            }

            // Mod State
            window._csMod = {
                active: true,
                mode: "twopass", // "twopass", "wireframe", "red", "blue"
                minCount: 1000,  // Cuts out small debris, particles, attachments
                maxCount: 4200   // Cuts out walls, floors, large level geometry
            };

            // --- 2. WEBGL VISUALS & FILTERING ---
            gl.drawElements = function(mode, count, type, offset) {
                // Ignore map geometry and small particles
                var isCharacter = (count >= window._csMod.minCount && count <= window._csMod.maxCount);

                if (window._csMod.active && isCharacter) {
                    if (window._csMod.mode === "twopass") {
                        // Pass 1: Hidden Behind Walls -> Pure Red
                        gl.depthFunc(gl.GREATER);
                        gl.colorMask(true, false, false, true);
                        window._origDrawElements.call(this, mode, count, type, offset);

                        // Pass 2: Visible in Line-of-Sight -> Pure Green
                        gl.depthFunc(gl.LEQUAL);
                        gl.colorMask(false, true, false, true);
                        window._origDrawElements.call(this, mode, count, type, offset);

                        // Reset states
                        gl.colorMask(true, true, true, true);
                        gl.depthFunc(gl.LEQUAL);
                    } 
                    else if (window._csMod.mode === "wireframe") {
                        // Convert 3D solid mesh into a wireframe skeleton
                        gl.disable(gl.DEPTH_TEST);
                        gl.colorMask(false, true, false, true); // Green wireframe
                        window._origDrawElements.call(this, gl.LINES, count, type, offset);
                        gl.colorMask(true, true, true, true);
                        gl.enable(gl.DEPTH_TEST);
                    } 
                    else if (window._csMod.mode === "red" || window._csMod.mode === "blue") {
                        gl.disable(gl.DEPTH_TEST);
                        if (window._csMod.mode === "red") gl.colorMask(true, false, false, true);
                        if (window._csMod.mode === "blue") gl.colorMask(false, false, true, true);
                        window._origDrawElements.apply(this, arguments);
                        gl.colorMask(true, true, true, true);
                        gl.enable(gl.DEPTH_TEST);
                    }
                } else {
                    window._origDrawElements.apply(this, arguments);
                }
            };

            // --- 3. ON-SCREEN INTERACTIVE MENU ---
            var oldMenu = document.getElementById("cs-mod-menu");
            if (oldMenu) oldMenu.remove();

            var menu = document.createElement("div");
            menu.id = "cs-mod-menu";
            menu.style.position = "fixed";
            menu.style.top = "15px";
            menu.style.right = "15px";
            menu.style.background = "rgba(10, 15, 20, 0.9)";
            menu.style.border = "2px solid #00ffaa";
            menu.style.padding = "10px";
            menu.style.color = "#fff";
            menu.style.fontFamily = "monospace";
            menu.style.zIndex = "999999";
            menu.style.borderRadius = "8px";
            menu.style.fontSize = "11px";
            menu.style.minWidth = "170px";
            menu.style.boxShadow = "0 0 10px rgba(0,255,170,0.3)";

            menu.innerHTML = `
                <div style="font-weight:bold; color:#00ffaa; text-align:center; margin-bottom:6px;">CS 1.6 ADVANCED MOD</div>
                <button id="btn-toggle" style="width:100%; margin:2px 0; background:#1a2228; color:#0fa; border:1px solid #345; cursor:pointer;">Visuals: ON</button>
                <button id="btn-mode" style="width:100%; margin:2px 0; background:#1a2228; color:#ffdd55; border:1px solid #345; cursor:pointer;">Mode: 2-Pass (R/G)</button>
                <button id="btn-speed" style="width:100%; margin:2px 0; background:#1a2228; color:#55ffff; border:1px solid #345; cursor:pointer;">Speed: 1.0x</button>
                
                <div style="margin-top:6px; font-size:10px; color:#aaa;">Mesh Min Filter:</div>
                <div style="display:flex; justify-content:space-between; align-items:center;">
                    <button id="btn-min-down" style="background:#222; color:#fff; border:1px solid #444; width:30px; cursor:pointer;">-</button>
                    <span id="lbl-min" style="color:#fff;">1000</span>
                    <button id="btn-min-up" style="background:#222; color:#fff; border:1px solid #444; width:30px; cursor:pointer;">+</button>
                </div>

                <div style="margin-top:4px; font-size:10px; color:#aaa;">Mesh Max Filter:</div>
                <div style="display:flex; justify-content:space-between; align-items:center;">
                    <button id="btn-max-down" style="background:#222; color:#fff; border:1px solid #444; width:30px; cursor:pointer;">-</button>
                    <span id="lbl-max" style="color:#fff;">4200</span>
                    <button id="btn-max-up" style="background:#222; color:#fff; border:1px solid #444; width:30px; cursor:pointer;">+</button>
                </div>
            `;
            document.body.appendChild(menu);

            // Button Event Handlers
            document.getElementById("btn-toggle").onclick = function() {
                window._csMod.active = !window._csMod.active;
                this.innerText = "Visuals: " + (window._csMod.active ? "ON" : "OFF");
                this.style.color = window._csMod.active ? "#0fa" : "#888";
            };

            document.getElementById("btn-mode").onclick = function() {
                if (window._csMod.mode === "twopass") {
                    window._csMod.mode = "wireframe";
                    this.innerText = "Mode: Wireframe";
                } else if (window._csMod.mode === "wireframe") {
                    window._csMod.mode = "red";
                    this.innerText = "Mode: Red Chams";
                } else if (window._csMod.mode === "red") {
                    window._csMod.mode = "blue";
                    this.innerText = "Mode: Blue Chams";
                } else {
                    window._csMod.mode = "twopass";
                    this.innerText = "Mode: 2-Pass (R/G)";
                }
            };

            document.getElementById("btn-speed").onclick = function() {
                if (window._gameSpeed === 1.0) window._gameSpeed = 1.4;
                else if (window._gameSpeed === 1.4) window._gameSpeed = 2.0;
                else if (window._gameSpeed === 2.0) window._gameSpeed = 0.5;
                else window._gameSpeed = 1.0;
                this.innerText = "Speed: " + window._gameSpeed.toFixed(1) + "x";
            };

            // Mesh Range Tuning Buttons
            document.getElementById("btn-min-down").onclick = function() {
                window._csMod.minCount = Math.max(0, window._csMod.minCount - 200);
                document.getElementById("lbl-min").innerText = window._csMod.minCount;
            };
            document.getElementById("btn-min-up").onclick = function() {
                window._csMod.minCount += 200;
                document.getElementById("lbl-min").innerText = window._csMod.minCount;
            };
            document.getElementById("btn-max-down").onclick = function() {
                window._csMod.maxCount = Math.max(window._csMod.minCount, window._csMod.maxCount - 300);
                document.getElementById("lbl-max").innerText = window._csMod.maxCount;
            };
            document.getElementById("btn-max-up").onclick = function() {
                window._csMod.maxCount += 300;
                document.getElementById("lbl-max").innerText = window._csMod.maxCount;
            };

            console.log("%c[Mod Ready] Advanced CS 1.6 Menu loaded.", "color: #00ffaa; font-weight: bold; font-size: 13px;");
        }
    }
})();
