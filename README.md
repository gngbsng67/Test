
(function() {
    // 1. Get the WebGL context from the Unity canvas
    const canvas = document.querySelector("#unity-canvas") || document.querySelector("canvas");
    const gl = canvas.getContext("webgl2") || canvas.getContext("webgl");

    if (!gl) {
        console.error("WebGL context not found!");
        return;
    }

    const originalDrawElements = gl.drawElements;

    // Set of index counts that belong to the character models 
    // (You determine these by logging 'count' and seeing which ones disappear/appear with bots)
    const targetMeshCounts = new Set([/* e.g., 1420, 3128 */]);

    // Optional: Log counts to find the model's signature
    let logDrawCalls = false;

    gl.drawElements = function(mode, count, type, offset) {
        if (logDrawCalls) {
            console.log("DrawElements count:", count);
        }

        // If this draw call matches a character mesh
        if (targetMeshCounts.has(count)) {
            // Disable depth test so it renders through walls
            gl.disable(gl.DEPTH_TEST);

            // Execute draw call (visible through geometry)
            originalDrawElements.apply(this, arguments);

            // Re-enable depth test for the rest of the scene
            gl.enable(gl.DEPTH_TEST);
            return;
        }

        return originalDrawElements.apply(this, arguments);
    };

    console.log("WebGL hook active. Inspect draw calls or populate targetMeshCounts.");
})();
