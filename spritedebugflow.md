# Sprite Debug Flow - Universal Guide for Fixing Sprite Animation Issues

## When Sprite Animation is Not Working, Follow This Flow:

### Step 1: Create Debug HTML File
- Create `[sprite_name_debug.html]` in your project root/public directory
- Copy the complete template from `mikedebug.html` below
- Update the sprite source path to point to your sprite

### Step 2: Configure Sprite Parameters
Update the sprite configuration section with your sprite's actual dimensions:

```javascript
const spriteConfig = {
    frameCount: [TOTAL_FRAMES],          // Number of frames in your sprite sheet
    currentFrame: 0,
    frameWidth: [SPRITE_WIDTH],        // Width of each frame in pixels
    frameHeight: [SPRITE_HEIGHT],      // Height of each frame in pixels
    columns: [COLUMNS],                // Number of columns in grid
    rows: [ROWS],                      // Number of rows in grid
    frameDelay: 75,                    // Animation speed (ms per frame)
    cropAmount: 0,                     // Initial crop amount
    spriteScale: 0.5                  // Initial size scale
};
```

### Step 3: Debug Common Issues

#### **Issue A: Scrolling Animation (Frames not properly separated)**
**Cause**: Incorrect frameWidth/frameHeight or grid dimensions
**Fix**:
1. Check your sprite sheet dimensions
2. Calculate: `frameWidth = spriteSheetWidth / columns`
3. Calculate: `frameHeight = spriteSheetHeight / rows`
4. Update spriteConfig with correct values

#### **Issue B: Wrong Frame Extraction (Frames shifted or overlapped)**
**Cause**: Incorrect grid layout or frame positioning
**Fix**:
1. Use arrow keys to step through each frame
2. Check if frame boundaries match expected positions
3. Adjust columns/rows values if needed
4. Verify frameCount matches actual frames

#### **Issue C: Sprite is Cropped or Stretched**
**Cause**: Wrong aspect ratio or destination dimensions
**Fix**:
1. Check source vs destination rectangles
2. Ensure proportional scaling
3. Adjust crop amounts if needed

### Step 4: Crop Adjustment Options

The debug tool provides 4 crop options:

#### **Crop from Top**: Removes pixels from top of each frame
```javascript
// In drawImage call:
pos.y + cropAmount,  // Source y (cropped from top)
spriteConfig.frameHeight - cropAmount,  // Source height
```

#### **Crop from Right**: Removes pixels from right of each frame
```javascript
// In drawImage call:
spriteConfig.frameWidth - cropAmount,  // Source width
```

#### **Crop from Bottom**: Removes pixels from bottom of each frame
```javascript
// In drawImage call:
spriteConfig.frameHeight - cropAmount,  // Source height
```

#### **Crop from Left**: Removes pixels from left of each frame
```javascript
// In drawImage call:
pos.x + cropAmount,  // Source x (cropped from left)
spriteConfig.frameWidth - cropAmount,  // Source width
```

### Step 5: Dynamic Frame-Specific Cropping

For sprites that need different crops for different frames:

```javascript
// Calculate dynamic crop amount based on frame
let dynamicCropAmount = 0;
const currentFrame = spriteConfig.currentFrame;

// Example: Different crops for different frame ranges
if (currentFrame >= 12 && currentFrame <= 17) {
    dynamicCropAmount = 3;  // 3px crop
} else if (currentFrame >= 18 && currentFrame <= 23) {
    dynamicCropAmount = 5;  // 5px crop
} else if (currentFrame >= 24 && currentFrame <= 35) {
    dynamicCropAmount = 10; // 10px crop
}
```

### Step 6: Testing Process

1. **Manual Frame Stepping**: Use arrow keys (← →) to step through each frame
2. **Animation Check**: Click "Play Animation" to see full loop
3. **Boundary Verification**: Check red overlay shows correct sprite boundaries
4. **Position Verification**: Ensure each frame shows the correct content
5. **Crop Adjustment**: Use sliders to adjust crop amounts in real-time

### Step 7: Apply Fix to Main Code

Once you've identified the correct configuration:

1. **Update frame dimensions** in your main sprite configuration
2. **Apply the same crop logic** to your game's drawImage calls
3. **Test in game** to ensure animation works correctly

### Debug HTML Template (Copy to `[sprite_name_debug.html]`):

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>[Sprite Name] Debug</title>
    <style>
        body {
            margin: 0;
            padding: 20px;
            background: #1a1a1a;
            color: white;
            font-family: Arial, sans-serif;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
        }
        .controls {
            background: #2a2a2a;
            padding: 20px;
            border-radius: 8px;
            margin-bottom: 20px;
        }
        .sprite-display {
            background: #333;
            padding: 20px;
            border-radius: 8px;
            text-align: center;
            min-height: 400px;
            display: flex;
            align-items: center;
            justify-content: center;
            flex-direction: column;
        }
        .canvas-container {
            background: white;
            border-radius: 8px;
            padding: 10px;
            margin: 20px 0;
        }
        canvas {
            display: block;
        }
        button {
            background: #4CAF50;
            color: white;
            border: none;
            padding: 10px 20px;
            margin: 5px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover {
            background: #45a049;
        }
        button:disabled {
            background: #666;
            cursor: not-allowed;
        }
        .info {
            background: #444;
            padding: 10px;
            border-radius: 4px;
            margin: 10px 0;
            font-family: monospace;
        }
        .frame-display {
            font-size: 24px;
            margin: 10px 0;
            color: #4CAF50;
        }
        .crop-options {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 10px;
            margin: 20px 0;
        }
        .crop-option {
            background: #555;
            padding: 10px;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>[Sprite Name] Debug Viewer</h1>

        <div class="controls">
            <h3>Sprite Configuration</h3>
            <div class="info">
                Sprite Sheet: [WIDTH]x[HEIGHT]px<br>
                Frame Count: [TOTAL_FRAMES]<br>
                Columns: [COLUMNS]<br>
                Rows: [ROWS]<br>
                Frame Width: [FRAME_WIDTH]px<br>
                Frame Height: [FRAME_HEIGHT]px
            </div>

            <div>
                <button id="prevBtn">← Previous</button>
                <button id="nextBtn">Next →</button>
                <button id="playBtn">Play Animation</button>
                <button id="stopBtn">Stop</button>
            </div>

            <div class="frame-display">
                Frame: <span id="frameNumber">0</span> / [TOTAL_FRAMES]
            </div>

            <div class="info">
                Frame Position: <span id="framePosition">0, 0</span><br>
                Source Rectangle: <span id="sourceRect">0, 0, [WIDTH], [HEIGHT]</span>
            </div>

            <div class="crop-options">
                <div class="crop-option">
                    <label>Crop from Top: <input type="range" id="cropTopSlider" min="0" max="50" value="0"></label>
                    <span id="cropTopValue">0</span>px
                </div>
                <div class="crop-option">
                    <label>Crop from Right: <input type="range" id="cropRightSlider" min="0" max="50" value="0"></label>
                    <span id="cropRightValue">0</span>px
                </div>
                <div class="crop-option">
                    <label>Crop from Bottom: <input type="range" id="cropBottomSlider" min="0" max="50" value="0"></label>
                    <span id="cropBottomValue">0</span>px
                </div>
                <div class="crop-option">
                    <label>Crop from Left: <input type="range" id="cropLeftSlider" min="0" max="50" value="0"></label>
                    <span id="cropLeftValue">0</span>px
                </div>
            </div>

            <div>
                <label>Sprite Size: <input type="range" id="sizeSlider" min="20" max="100" value="50"></label>
                <span id="sizeValue">50</span>%
            </div>
        </div>

        <div class="sprite-display">
            <div class="canvas-container">
                <canvas id="spriteCanvas" width="300" height="300"></canvas>
            </div>
        </div>
    </div>

    <script>
        // UPDATE THESE VALUES FOR YOUR SPRITE
        const spriteConfig = {
            frameCount: [TOTAL_FRAMES],
            currentFrame: 0,
            frameWidth: [FRAME_WIDTH],
            frameHeight: [FRAME_HEIGHT],
            columns: [COLUMNS],
            rows: [ROWS],
            frameDelay: 75,
            cropTop: 0,
            cropRight: 0,
            cropBottom: 0,
            cropLeft: 0,
            spriteScale: 0.5
        };

        let isPlaying = false;
        let animationId = null;
        let spriteImage = new Image();
        let spriteLoaded = false;
        let lastFrameTime = 0;

        // UPDATE SPRITE SOURCE PATH
        spriteImage.src = '/[sprite_name].png';

        spriteImage.onload = () => {
            spriteLoaded = true;
            console.log('Sprite loaded:', spriteImage.naturalWidth + 'x' + spriteImage.naturalHeight);
            drawFrame();
        };

        spriteImage.onerror = () => {
            console.error('Failed to load sprite');
            const canvas = document.getElementById('spriteCanvas');
            const ctx = canvas.getContext('2d');
            ctx.fillStyle = 'red';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = 'white';
            ctx.font = '20px Arial';
            ctx.fillText('Failed to load sprite', 10, 30);
        };

        // Rest of the JavaScript code from mikedebug.html...
        // (Include all the calculateFramePosition, drawFrame, animate functions, and event listeners)
    </script>
</body>
</html>
```

### Quick Reference Commands:

- **Arrow Keys**: ← Previous frame, → Next frame
- **Spacebar**: Toggle play/stop animation
- **Sliders**: Adjust crop amounts in real-time
- **Frame Counter**: Shows current frame number
- **Source Rectangle**: Shows exact drawImage parameters

### Common Problems & Solutions:

| Problem | Cause | Solution |
|---------|--------|----------|
| Scrolling instead of individual frames | Wrong frameWidth/frameHeight | Calculate from sprite dimensions |
| Frames shifted/offset | Wrong columns/rows count | Verify grid layout matches sprite |
| Black screens or missing frames | Wrong sprite path or frame count | Check file exists and has correct frames |
| Stretching/distortion | Wrong aspect ratio | Maintain proportional scaling |

---

**Usage**: Tell Claude "follow spritedebugflow.md to debug [sprite_name]" and it will guide you through this complete debugging process.