(() => {
    // 1. Locate the game's WebGL canvas
    const canvas = document.querySelector("#unity-canvas") || document.querySelector("canvas");
    if (!canvas) {
        console.error("[Mod] Could not find the Unity canvas element.");
        return;
    }

    const gl = canvas.getContext("webgl2") || canvas.getContext("webgl");
    if (!gl) {
        console.error("[Mod] Could not get WebGL context.");
        return;
    }

    // 2. Prevent stacking multiple hooks if run repeatedly
    if (!window._originalDrawElements) {
        window._originalDrawElements = gl.drawElements;
    }

    // Configuration state
    window._espState = {
        enabled: true,         // Toggle see-through walls
        logCounts: false,      // Log mesh index counts to console
        filterByCount: false,  // Set to true once you know the bot model counts
        targetCounts: new Set([]) // Put your bot model numbers here, e.g. [1420, 2800]
    };

    // 3. Hook the draw call
    gl.drawElements = function(mode, count, type, offset) {
        if (window._espState.logCounts) {
            console.log("Mesh Count:", count);
        }

        if (window._espState.enabled) {
            // Mode A: Target specific bot meshes (if targetCounts is set)
            if (window._espState.filterByCount) {
                if (window._espState.targetCounts.has(count)) {
                    gl.disable(gl.DEPTH_TEST);
                    window._originalDrawElements.apply(this, arguments);
                    gl.enable(gl.DEPTH_TEST);
                    return;
                }
            } 
            // Mode B: Universal test mode (ignores depth on 3D triangle meshes)
            else if (mode === gl.TRIANGLES && count > 100) {
                gl.disable(gl.DEPTH_TEST);
                window._originalDrawElements.apply(this, arguments);
                gl.enable(gl.DEPTH_TEST);
                return;
            }
        }

        return window._originalDrawElements.apply(this, arguments);
    };

    // 4. Keyboard Shortcuts
    window.addEventListener("keydown", (e) => {
        // Press 'B' to toggle the see-through effect on/off
        if (e.key === "b" || e.key === "B") {
            window._espState.enabled = !window._espState.enabled;
            console.log("[Mod] See-through mode:", window._espState.enabled ? "ON" : "OFF");
        }
        // Press 'L' to toggle logging mesh counts to console
        if (e.key === "l" || e.key === "L") {
            window._espState.logCounts = !window._espState.logCounts;
            console.log("[Mod] Logging mesh counts:", window._espState.logCounts ? "ON" : "OFF");
        }
    });

    console.log("%c[Mod] WebGL Hook Successfully Injected!", "color: #00ff00; font-weight: bold;");
    console.log("• Press [B] to toggle See-Through mode on/off.");
    console.log("• Press [L] to toggle logging mesh counts in console.");
})();
