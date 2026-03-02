/**
 * CM2030 - Graphics Programming CW2
 * Justin Kwek SIM ID 10274652
 *
 * ============================================================
 APPLICATION WALKTHROUGH
The application is split into two pages which are the Main App and the Thermal Imaging Extension respectively. The Main App captures live webcam footage and routes it through five processing groups. Group 1 converts the feed to greyscale with brightness reduced by 20%, using the luminance formula inside a single pixel loop with a clamp to prevent sub-zero values. Group 2 isolates each RGB channel. Group 3 applies binary thresholding per channel via sliders. Group 4 converts the frame to HSV and YCbCr using standard matrix formulae. Group 5 handles face detection via ml5 faceApi, colour space thresholding, and privacy filters applied directly to the detected face region. The Thermal page offers four modes: Basic (luminance-mapped palette), Motion Heat (frame differencing with a persistent heat map), Body Heat (face-region boost with skin-tone detection), and Edge Detection (Sobel operator with a neon palette).

PROBLEMS FACED
The largest challenge was performance because rendering all fourteen preview canvases causes the frame rate to drop significantly. The solution to this was a group-toggle system where only the currently active groups are processed and rendered each frame, reducing per-frame pixel operations substantially. Groups can be toggled on and off individually using keys 1–5, allowing any combination from a single group to all five simultaneously. Another issue involved stale ml5 detections persisting after a webcam reset, resolved by clearing the detections array and nullifying the faceapi instance before reinitialising. A third issue is with the horizontal flip was pixels being overwritten mid-copy. This was fixed by buffering each row before writing it back in reverse.

ON TARGET?
All core tasks were completed successfully. The thermal extension is a welcome addition with four distinct modes and full face-tracking overlays utilising ML5JS face tracking features and techniques taught throughout the module.

EXTENSION DISCUSSION
The Thermal Imaging extension combines frame differencing, a persistent heat map with exponential cooling, particle trail rendering, multi-stop palette interpolation, edge detection, and live face-tracking with skin-tone temperature estimation. This is possible by using a photoresister/light sensor that every web camera has.
 * ============================================================
 */

// ============================================
// PAGE MANAGEMENT
// ============================================
let currentPage = 0; // 0 = Main, 1 = Thermal

// ============================================
// GROUP TAB MANAGEMENT
// ============================================
// Groups: 0=Basic Ops, 1=RGB Channels, 2=RGB Threshold, 3=Colour Spaces, 4=Advanced
// activeGroups is a Set — multiple groups can be visible simultaneously.
// Pressing a key (1–5) toggles that group on/off.
let activeGroups = new Set([0]); // Start with group 0 open
const GROUP_COUNT = 5;

// ============================================
// MAIN APPLICATION VARIABLES
// ============================================
let capture;
let canvas;
let snapshot = null;
let currentImage = null;
let gridImages = {};
let sliders = {};
let faceFilterType = 'none';
let pixelBlockSize = 5;
let webcamActive = false;

// Face detection with ml5
let faceapi;
let detections = [];
let modelLoaded = false;

// Image dimensions
const w = 320;
const h = 240;
const THERMAL_WIDTH = 800;
const THERMAL_HEIGHT = 600;

// UI elements references
let previewElements = {};

// ============================================
// THERMAL IMAGING VARIABLES
// ============================================
let thermalGraphics;
let previousFrame;
let heatMap = [];
let heatTrails = [];

// Thermal modes
const MODES = {
  BASIC: 0,
  MOTION_HEAT: 1,
  BODY_HEAT: 2,
  EDGE_DETECTION: 3
};

let currentThermalMode = MODES.BASIC;

// Toggle states
let thermalToggles = {
  showFaceBoxes: true,
  showLandmarks: true,
  showTemperatureLabels: true,
  showFaceHeat: true
};

// Thermal colour palette
const THERMAL_PALETTE = [
  [0, 0, 100],
  [0, 100, 200],
  [0, 200, 255],
  [100, 255, 100],
  [255, 255, 0],
  [255, 150, 0],
  [255, 0, 0],
  [255, 255, 255]
];

// Edge detection specific colours
const EDGE_COLORS = {
  background: [20, 20, 40],
  edge: [0, 255, 255],
  strongEdge: [255, 0, 255]
};

// Configuration
let thermalConfig = {
  temperatureScale: 1.0,
  faceHeatBoost: 50,
  coolingRate: 0.95,
  motionSensitivity: 0.15,
  edgeThreshold: 30,
  edgeStrength: 1.5
};

// Statistics
let thermalStats = {
  faceCount: 0,
  faceTemp: 25,
  edgeCount: 0
};

// Thermal UI Elements
let thermalModeButtons = [];
let thermalToggleButtons = [];
let thermalStatusDiv;

// ============================================
// SETUP FUNCTION
// ============================================
function setup() {
    console.log('Setting up application...');

    canvas = createCanvas(THERMAL_WIDTH, THERMAL_HEIGHT);
    canvas.parent('canvas-container');
    pixelDensity(1);

    initWebcam();
    initThermal();
    initFaceDetection();
    setupPreviewElements();
    setupControls();
    setupButtons();
    setupThermalUI();
    setupGroupTabs();
    initializeGridImages();

    window.switchPage = switchPage;

    console.log('Setup complete!');
}

// ============================================
// GROUP TAB SYSTEM
// ============================================
function setupGroupTabs() {
    const tabBar = document.getElementById('group-tab-bar');
    if (!tabBar) return;

    const tabLabels = [
        '📷 1: Basic Ops',
        '🎨 2: RGB Channels',
        '⚫ 3: Thresholding',
        '🌈 4: Colour Spaces',
        '👤 5: Advanced'
    ];

    tabBar.innerHTML = '';
    tabLabels.forEach((label, i) => {
        const btn = document.createElement('button');
        btn.className = 'group-tab-btn' + (activeGroups.has(i) ? ' active-tab' : '');
        btn.textContent = label;
        btn.addEventListener('click', () => toggleGroup(i));
        tabBar.appendChild(btn);
    });

    updateGroupVisibility();
}

function toggleGroup(groupIndex) {
    if (activeGroups.has(groupIndex)) {
        // Only allow closing if at least one other group stays open
        if (activeGroups.size > 1) {
            activeGroups.delete(groupIndex);
        }
    } else {
        activeGroups.add(groupIndex);
    }

    // Sync button styles
    const tabBar = document.getElementById('group-tab-bar');
    if (tabBar) {
        Array.from(tabBar.children).forEach((btn, i) => {
            btn.classList.toggle('active-tab', activeGroups.has(i));
        });
    }

    updateGroupVisibility();
}

function updateGroupVisibility() {
    const groupIds = ['group-1','group-2','group-3','group-4','group-5'];
    groupIds.forEach((id, i) => {
        const el = document.getElementById(id);
        if (el) el.style.display = activeGroups.has(i) ? 'block' : 'none';
    });
}

// ============================================
// PAGE MANAGEMENT
// ============================================
function switchPage(pageNum) {
    currentPage = pageNum;
    if (pageNum === 0) {
        canvas.parent('canvas-container');
    } else {
        canvas.parent('thermal-canvas-container');
    }
}

// ============================================
// INITIALIZATION FUNCTIONS
// ============================================
function initWebcam() {
    try {
        if (capture) {
            capture.remove();
            capture = null;
        }
        
        capture = createCapture(VIDEO);
        capture.size(w, h);
        capture.hide();
        webcamActive = true;
        
        detections = [];
        thermalStats.faceCount = 0;
        
        select('#webcam-status').html('Webcam: Active');
        console.log('Webcam initialized');
        
        if (!faceapi) {
            initFaceDetection();
        }
    } catch (err) {
        console.error('Error initializing webcam:', err);
        createPlaceholderImage();
        select('#webcam-status').html('Webcam: Failed - Using Test Pattern');
    }
}

function initThermal() {
    thermalGraphics = createGraphics(THERMAL_WIDTH, THERMAL_HEIGHT);
    previousFrame   = createGraphics(w, h);

    const totalPixels = w * h;
    heatMap = new Array(totalPixels).fill(0);
    heatTrails = [];
}

function initFaceDetection() {
    thermalStatusDiv = select('#thermal-status');
    thermalStatusDiv.html('Loading face detection model...');
    
    if (!capture) {
        console.log('Waiting for capture before initializing face detection');
        setTimeout(initFaceDetection, 500);
        return;
    }
    
    const detectionOptions = {
        withLandmarks: true,
        withDescriptors: false,
        minConfidence: 0.5
    };
    
    detections = [];
    faceapi = ml5.faceApi(capture, detectionOptions, modelReady);
}

function modelReady() {
    modelLoaded = true;
    thermalStatusDiv.html('Face detection ready!');
    console.log('Face detection model ready');
    
    detections = [];
    thermalStats.faceCount = 0;
    
    detectFaces();
}

function detectFaces() {
    if (modelLoaded && faceapi && capture && capture.width > 0 && capture.height > 0) {
        faceapi.detect(gotFaces);
    } else {
        setTimeout(detectFaces, 100);
    }
}

function gotFaces(error, results) {
    if (error) {
        console.error('Face detection error:', error);
        thermalStatusDiv.html('Error detecting faces');
        setTimeout(detectFaces, 1000);
    } else {
        if (results) {
            detections = results;
            thermalStats.faceCount = detections.length;
            select('#face-status').html(`Faces: ${detections.length} detected`);
            
            if (detections.length > 0) {
                thermalStats.faceTemp = calculateFaceTemperature(detections[0]);
            }
        }
        setTimeout(detectFaces, 100);
    }
}

function createPlaceholderImage() {
    console.log('Creating placeholder image');
    currentImage = createImage(w, h);
    currentImage.loadPixels();
    for (let x = 0; x < w; x++) {
        for (let y = 0; y < h; y++) {
            let index = (x + y * w) * 4;
            currentImage.pixels[index]     = (x / w) * 255;
            currentImage.pixels[index + 1] = (y / h) * 255;
            currentImage.pixels[index + 2] = 128 + 127 * sin(x * 0.05) * cos(y * 0.05);
            currentImage.pixels[index + 3] = 255;
        }
    }
    currentImage.updatePixels();
    snapshot = currentImage;
}

function setupPreviewElements() {
    previewElements = {
        webcam:       select('#webcam-preview'),
        grayscale:    select('#grayscale-preview'),
        redChannel:   select('#red-preview'),
        greenChannel: select('#green-preview'),
        blueChannel:  select('#blue-preview'),
        thresholdR:   select('#thresholdR-preview'),
        thresholdG:   select('#thresholdG-preview'),
        thresholdB:   select('#thresholdB-preview'),
        webcam2:      select('#webcam2-preview'),
        colourSpace1: select('#cs1-preview'),
        colourSpace2: select('#cs2-preview'),
        faceDetection:select('#face-preview'),
        thresholdCS1: select('#thresholdCS1-preview'),
        thresholdCS2: select('#thresholdCS2-preview')
    };

    for (let key in previewElements) {
        if (previewElements[key]) {
            previewElements[key].html('');
            previewElements[key].style('background-color', '#1a1a2e');
        }
    }
}

function setupControls() {
    sliders.thresholdR   = select('#slider-r');
    sliders.thresholdG   = select('#slider-g');
    sliders.thresholdB   = select('#slider-b');
    sliders.thresholdCS1 = select('#slider-cs1');
    sliders.thresholdCS2 = select('#slider-cs2');

    if (sliders.thresholdR)   sliders.thresholdR.input(()   => select('#r-value').html(sliders.thresholdR.value()));
    if (sliders.thresholdG)   sliders.thresholdG.input(()   => select('#g-value').html(sliders.thresholdG.value()));
    if (sliders.thresholdB)   sliders.thresholdB.input(()   => select('#b-value').html(sliders.thresholdB.value()));
    if (sliders.thresholdCS1) sliders.thresholdCS1.input(() => select('#cs1-value').html(sliders.thresholdCS1.value()));
    if (sliders.thresholdCS2) sliders.thresholdCS2.input(() => select('#cs2-value').html(sliders.thresholdCS2.value()));
}

function setupButtons() {
    select('#btn-snapshot')?.mousePressed(captureSnapshot);
    select('#btn-reset')?.mousePressed(resetWebcam);

    select('#btn-gray')?.mousePressed(()     => setFaceFilter('grayscale'));
    select('#btn-flip')?.mousePressed(()     => setFaceFilter('flip'));
    select('#btn-colour')?.mousePressed(()   => setFaceFilter('colour'));
    select('#btn-pixelate')?.mousePressed(() => setFaceFilter('pixelate'));
}

function setupThermalUI() {
    const modeContainer = select('#thermal-mode-buttons');
    const modeNames = ['Basic', 'Motion', 'Body Heat', 'Edge'];

    for (let i = 0; i < modeNames.length; i++) {
        const btn = createButton(modeNames[i]);
        btn.parent(modeContainer);
        btn.class('thermal-mode-btn');
        btn.mousePressed(() => setThermalMode(i));
        thermalModeButtons.push(btn);
    }

    const toggleContainer = select('#thermal-toggle-buttons');
    const toggleConfigs = [
        { id: 'showFaceBoxes',          label: 'Face Boxes', color: '#ff6b6b' },
        { id: 'showLandmarks',          label: 'Landmarks',  color: '#ffaa00' },
        { id: 'showTemperatureLabels',  label: 'Temp Labels',color: '#ff00ff' },
        { id: 'showFaceHeat',           label: 'Face Heat',  color: '#ff8800' }
    ];

    for (let cfg of toggleConfigs) {
        const btn = createButton(cfg.label);
        btn.parent(toggleContainer);
        btn.class('thermal-toggle-btn');
        if (thermalToggles[cfg.id]) btn.addClass('active');
        btn.style('border-color', cfg.color);
        btn.mousePressed(() => toggleThermalFeature(cfg.id, btn));
        thermalToggleButtons.push(btn);
    }

    setThermalMode(0);

    setupThermalSlider('temp-scale', 'temperatureScale', 'temp-value', v => `${v}x`);
    setupThermalSlider('face-heat',  'faceHeatBoost',    'heat-value', v => `${v}%`);
}

function setupThermalSlider(id, configKey, valueId, formatter) {
    const slider    = select(`#${id}`);
    const valueSpan = select(`#${valueId}`);
    slider.input(() => {
        thermalConfig[configKey] = parseFloat(slider.value());
        valueSpan.html(formatter(slider.value()));
    });
}

function setThermalMode(mode) {
    currentThermalMode = mode;
    for (let i = 0; i < thermalModeButtons.length; i++) {
        if (i === mode) thermalModeButtons[i].addClass('active');
        else            thermalModeButtons[i].removeClass('active');
    }
}

function toggleThermalFeature(featureId, button) {
    thermalToggles[featureId] = !thermalToggles[featureId];
    if (thermalToggles[featureId]) button.addClass('active');
    else                           button.removeClass('active');
}

function initializeGridImages() {
    gridImages = {
        webcam:       createImage(w, h),
        grayscale:    createImage(w, h),
        redChannel:   createImage(w, h),
        greenChannel: createImage(w, h),
        blueChannel:  createImage(w, h),
        thresholdR:   createImage(w, h),
        thresholdG:   createImage(w, h),
        thresholdB:   createImage(w, h),
        webcam2:      createImage(w, h),
        colourSpace1: createImage(w, h),
        colourSpace2: createImage(w, h),
        thresholdCS1: createImage(w, h),
        thresholdCS2: createImage(w, h),
        faceDetection:createImage(w, h)
    };
}

function setFaceFilter(filterType) {
    faceFilterType = filterType;
    updateButtonStates();
    select('#filter-status').html(`Filter: ${filterType.toUpperCase()}`);
}

function updateButtonStates() {
    select('#btn-gray')?.removeClass('active-filter');
    select('#btn-flip')?.removeClass('active-filter');
    select('#btn-colour')?.removeClass('active-filter');
    select('#btn-pixelate')?.removeClass('active-filter');

    if (faceFilterType !== 'none') {
        const btnId = faceFilterType === 'grayscale' ? '#btn-gray' : 
                     faceFilterType === 'flip' ? '#btn-flip' :
                     faceFilterType === 'colour' ? '#btn-colour' :
                     faceFilterType === 'pixelate' ? '#btn-pixelate' : null;
        if (btnId) {
            select(btnId)?.addClass('active-filter');
        }
    }
}

// ============================================
// DRAW LOOP
// ============================================
function draw() {
    if (snapshot) {
        currentImage = snapshot;
    } else if (capture && capture.width > 0) {
        currentImage = capture;
    } else {
        return;
    }

    if (currentPage === 0) {
        drawMainPage();
    } else {
        drawThermalPage();
    }
}

function drawMainPage() {
    // Process and render every currently visible group
    for (let g of activeGroups) {
        processGroupImages(g);
        updateGroupPreviews(g);
    }
    updateStatus();
}

function drawThermalPage() {
    updateThermalDisplay();
}

// ============================================
// GROUP-BASED PROCESSING
// ============================================
function processGroupImages(group) {
    if (!currentImage) return;

    // Group 0: Basic Operations — webcam + grayscale
    if (group === 0) {
        gridImages.webcam.copy(currentImage, 0, 0, w, h, 0, 0, w, h);
        processGrayscaleBrightness();
    }

    // Group 1: RGB Channels
    if (group === 1) {
        gridImages.webcam.copy(currentImage, 0, 0, w, h, 0, 0, w, h);
        processRGBChannels();
    }

    // Group 2: RGB Thresholding
    if (group === 2) {
        gridImages.webcam.copy(currentImage, 0, 0, w, h, 0, 0, w, h);
        processRGBThresholding();
    }

    // Group 3: Colour Space Conversion
    if (group === 3) {
        gridImages.webcam.copy(currentImage, 0, 0, w, h, 0, 0, w, h);
        gridImages.webcam2.copy(gridImages.webcam, 0, 0, w, h, 0, 0, w, h);
        processColourSpaces();
    }

    // Group 4: Advanced — face detection + colour space thresholding
    if (group === 4) {
        gridImages.webcam.copy(currentImage, 0, 0, w, h, 0, 0, w, h);
        // Colour spaces needed for thresholding
        processColourSpaces();
        processColourSpaceThresholding();
        processFaceDetectionFilter();
    }
}

function updateGroupPreviews(group) {
    const groupKeys = [
        ['webcam', 'grayscale'],                          // group 0
        ['redChannel', 'greenChannel', 'blueChannel'],    // group 1
        ['thresholdR', 'thresholdG', 'thresholdB'],       // group 2
        ['webcam2', 'colourSpace1', 'colourSpace2'],       // group 3
        ['faceDetection', 'thresholdCS1', 'thresholdCS2'] // group 4
    ];

    const keys = groupKeys[group] || [];
    for (let key of keys) {
        if (previewElements[key] && gridImages[key]) {
            renderImageToPreview(gridImages[key], previewElements[key]);
        }
    }
}

function renderImageToPreview(img, element) {
    let tempCanvas = document.createElement('canvas');
    tempCanvas.width  = w;
    tempCanvas.height = h;
    let ctx = tempCanvas.getContext('2d');
    ctx.drawImage(img.canvas, 0, 0, w, h);
    let dataUrl = tempCanvas.toDataURL();
    element.style('background-image',    `url('${dataUrl}')`);
    element.style('background-size',     'cover');
    element.style('background-position', 'center');
}

function updatePreviews() {
    for (let key in gridImages) {
        if (previewElements[key] && gridImages[key]) {
            renderImageToPreview(gridImages[key], previewElements[key]);
        }
    }
}

function processGrayscaleBrightness() {
    let src  = gridImages.webcam;
    let dest = gridImages.grayscale;
    dest.loadPixels();
    src.loadPixels();
    for (let i = 0; i < src.pixels.length; i += 4) {
        let gray = (0.299 * src.pixels[i] + 0.587 * src.pixels[i+1] + 0.114 * src.pixels[i+2]) * 0.8;
        gray = max(gray, 0);
        dest.pixels[i] = dest.pixels[i+1] = dest.pixels[i+2] = gray;
        dest.pixels[i+3] = 255;
    }
    dest.updatePixels();
}

function processRGBChannels() {
    let src = gridImages.webcam;
    src.loadPixels();

    gridImages.redChannel.loadPixels();
    gridImages.greenChannel.loadPixels();
    gridImages.blueChannel.loadPixels();

    for (let i = 0; i < src.pixels.length; i += 4) {
        gridImages.redChannel.pixels[i]   = src.pixels[i]; gridImages.redChannel.pixels[i+1]   = 0; gridImages.redChannel.pixels[i+2]   = 0; gridImages.redChannel.pixels[i+3]   = 255;
        gridImages.greenChannel.pixels[i] = 0; gridImages.greenChannel.pixels[i+1] = src.pixels[i+1]; gridImages.greenChannel.pixels[i+2] = 0; gridImages.greenChannel.pixels[i+3] = 255;
        gridImages.blueChannel.pixels[i]  = 0; gridImages.blueChannel.pixels[i+1]  = 0; gridImages.blueChannel.pixels[i+2]  = src.pixels[i+2]; gridImages.blueChannel.pixels[i+3]  = 255;
    }

    gridImages.redChannel.updatePixels();
    gridImages.greenChannel.updatePixels();
    gridImages.blueChannel.updatePixels();
}

function processRGBThresholding() {
    let src = gridImages.webcam;
    if (sliders.thresholdR) thresholdChannel(src, gridImages.thresholdR, sliders.thresholdR.value(), 'red');
    if (sliders.thresholdG) thresholdChannel(src, gridImages.thresholdG, sliders.thresholdG.value(), 'green');
    if (sliders.thresholdB) thresholdChannel(src, gridImages.thresholdB, sliders.thresholdB.value(), 'blue');
}

function thresholdChannel(src, dest, threshold, channel) {
    dest.loadPixels();
    src.loadPixels();
    for (let i = 0; i < src.pixels.length; i += 4) {
        let value = channel === 'red' ? src.pixels[i] : channel === 'green' ? src.pixels[i+1] : src.pixels[i+2];
        let binary = value > threshold ? 255 : 0;
        dest.pixels[i] = dest.pixels[i+1] = dest.pixels[i+2] = binary;
        dest.pixels[i+3] = 255;
    }
    dest.updatePixels();
}

function processColourSpaces() {
    convertToHSV(gridImages.webcam, gridImages.colourSpace1);
    convertToYCbCr(gridImages.webcam, gridImages.colourSpace2);
}

function convertToHSV(src, dest) {
    dest.loadPixels();
    src.loadPixels();
    for (let i = 0; i < src.pixels.length; i += 4) {
        let r = src.pixels[i] / 255, g = src.pixels[i+1] / 255, b = src.pixels[i+2] / 255;
        let maxVal = max(r, g, b), minVal = min(r, g, b);
        let h = 0, s = maxVal === 0 ? 0 : (maxVal - minVal) / maxVal, v = maxVal;
        if (maxVal !== minVal) {
            if      (maxVal === r) h = 60 * ((g - b) / (maxVal - minVal));
            else if (maxVal === g) h = 60 * (2 + (b - r) / (maxVal - minVal));
            else                   h = 60 * (4 + (r - g) / (maxVal - minVal));
        }
        if (h < 0) h += 360;
        dest.pixels[i] = (h / 360) * 255;
        dest.pixels[i+1] = s * 255;
        dest.pixels[i+2] = v * 255;
        dest.pixels[i+3] = 255;
    }
    dest.updatePixels();
}

function convertToYCbCr(src, dest) {
    dest.loadPixels();
    src.loadPixels();
    for (let i = 0; i < src.pixels.length; i += 4) {
        let r = src.pixels[i], g = src.pixels[i+1], b = src.pixels[i+2];
        dest.pixels[i]   = constrain(0.299*r + 0.587*g + 0.114*b, 0, 255);
        dest.pixels[i+1] = constrain(128 - 0.168736*r - 0.331264*g + 0.5*b, 0, 255);
        dest.pixels[i+2] = constrain(128 + 0.5*r - 0.418688*g - 0.081312*b, 0, 255);
        dest.pixels[i+3] = 255;
    }
    dest.updatePixels();
}

function processColourSpaceThresholding() {
    if (sliders.thresholdCS1) simpleThreshold(gridImages.colourSpace1, gridImages.thresholdCS1, sliders.thresholdCS1.value());
    if (sliders.thresholdCS2) simpleThreshold(gridImages.colourSpace2, gridImages.thresholdCS2, sliders.thresholdCS2.value());
}

function simpleThreshold(src, dest, threshold) {
    dest.loadPixels();
    src.loadPixels();
    for (let i = 0; i < src.pixels.length; i += 4) {
        let binary = src.pixels[i] > threshold ? 255 : 0;
        dest.pixels[i] = dest.pixels[i+1] = dest.pixels[i+2] = binary;
        dest.pixels[i+3] = 255;
    }
    dest.updatePixels();
}

function processFaceDetectionFilter() {
    gridImages.faceDetection.copy(gridImages.webcam, 0, 0, w, h, 0, 0, w, h);
    if (detections && detections.length > 0) {
        for (let detection of detections) {
            let box = detection.alignedRect._box;
            if (box) {
                let x    = Math.max(0, floor(box._x));
                let y    = Math.max(0, floor(box._y));
                let faceW = Math.min(floor(box._width),  w - x);
                let faceH = Math.min(floor(box._height), h - y);
                applyFaceFilterToRegion(gridImages.faceDetection, x, y, faceW, faceH);
                drawFaceBoundingBox(gridImages.faceDetection, x, y, faceW, faceH);
            }
        }
    } else if (faceFilterType !== 'none') {
        applyFaceFilterToRegion(gridImages.faceDetection, 0, 0, w, h);
    }
}

function applyFaceFilterToRegion(img, x, y, fw, fh) {
    x  = Math.max(0, Math.min(x, img.width - 1));
    y  = Math.max(0, Math.min(y, img.height - 1));
    fw = Math.min(fw, img.width  - x);
    fh = Math.min(fh, img.height - y);
    if (fw <= 0 || fh <= 0) return;

    switch (faceFilterType) {
        case 'grayscale': applyGrayscaleToRegion(img, x, y, fw, fh);     break;
        case 'flip':      applyHorizontalFlipToRegion(img, x, y, fw, fh); break;
        case 'colour':    applyColourInvertToRegion(img, x, y, fw, fh);   break;
        case 'pixelate':  applyPixelateToRegion(img, x, y, fw, fh);       break;
    }
}

function applyGrayscaleToRegion(img, x, y, fw, fh) {
    img.loadPixels();
    for (let j = y; j < y + fh; j++) {
        for (let i = x; i < x + fw; i++) {
            let idx  = (i + j * img.width) * 4;
            let gray = 0.299 * img.pixels[idx] + 0.587 * img.pixels[idx+1] + 0.114 * img.pixels[idx+2];
            img.pixels[idx] = img.pixels[idx+1] = img.pixels[idx+2] = gray;
        }
    }
    img.updatePixels();
}

function applyHorizontalFlipToRegion(img, x, y, fw, fh) {
    img.loadPixels();
    // Buffer each row before writing back in reverse
    for (let j = y; j < y + fh; j++) {
        let rowBuffer = [];
        for (let i = x; i < x + fw; i++) {
            let idx = (i + j * img.width) * 4;
            rowBuffer.push({ r: img.pixels[idx], g: img.pixels[idx+1], b: img.pixels[idx+2], a: img.pixels[idx+3] });
        }
        for (let i = 0; i < fw; i++) {
            let idx = ((x + fw - 1 - i) + j * img.width) * 4;
            let px  = rowBuffer[i];
            img.pixels[idx] = px.r; img.pixels[idx+1] = px.g; img.pixels[idx+2] = px.b; img.pixels[idx+3] = px.a;
        }
    }
    img.updatePixels();
}

function applyColourInvertToRegion(img, x, y, fw, fh) {
    img.loadPixels();
    for (let j = y; j < y + fh; j++) {
        for (let i = x; i < x + fw; i++) {
            let idx = (i + j * img.width) * 4;
            img.pixels[idx]   = 255 - img.pixels[idx];
            img.pixels[idx+1] = 255 - img.pixels[idx+1];
            img.pixels[idx+2] = 255 - img.pixels[idx+2];
        }
    }
    img.updatePixels();
}

function applyPixelateToRegion(img, x, y, fw, fh) {
    applyGrayscaleToRegion(img, x, y, fw, fh);
    img.loadPixels();

    let grayValues = [];
    for (let j = y; j < y + fh; j++) {
        for (let i = x; i < x + fw; i++) {
            grayValues.push(img.pixels[(i + j * img.width) * 4]);
        }
    }

    // Clear region
    for (let j = y; j < y + fh; j++) {
        for (let i = x; i < x + fw; i++) {
            let idx = (i + j * img.width) * 4;
            img.pixels[idx] = img.pixels[idx+1] = img.pixels[idx+2] = 0;
        }
    }

    // Paint circles per block
    for (let blockY = 0; blockY < fh; blockY += pixelBlockSize) {
        for (let blockX = 0; blockX < fw; blockX += pixelBlockSize) {
            let total = 0, count = 0;
            for (let j = 0; j < pixelBlockSize && blockY + j < fh; j++) {
                for (let i = 0; i < pixelBlockSize && blockX + i < fw; i++) {
                    let localIdx = (blockX + i) + (blockY + j) * fw;
                    if (localIdx < grayValues.length) { total += grayValues[localIdx]; count++; }
                }
            }
            let avg     = count > 0 ? total / count : 0;
            let centreX = x + blockX + pixelBlockSize / 2;
            let centreY = y + blockY + pixelBlockSize / 2;
            let radius  = pixelBlockSize / 2;
            for (let dy = -radius; dy <= radius; dy++) {
                for (let dx = -radius; dx <= radius; dx++) {
                    if (dx*dx + dy*dy <= radius*radius) {
                        let imgX = Math.floor(centreX + dx), imgY = Math.floor(centreY + dy);
                        if (imgX >= x && imgX < x + fw && imgY >= y && imgY < y + fh &&
                            imgX >= 0 && imgX < img.width && imgY >= 0 && imgY < img.height) {
                            let idx = (imgX + imgY * img.width) * 4;
                            img.pixels[idx] = img.pixels[idx+1] = img.pixels[idx+2] = avg;
                        }
                    }
                }
            }
        }
    }
    img.updatePixels();
}

function drawFaceBoundingBox(img, x, y, fw, fh) {
    img.loadPixels();
    let thickness = 2;
    for (let i = x; i < x + fw; i++) {
        for (let t = 0; t < thickness; t++) {
            if (y + t < img.height) {
                let idx = (i + (y + t) * img.width) * 4;
                img.pixels[idx] = 255; img.pixels[idx+1] = 0; img.pixels[idx+2] = 0;
            }
            if (y + fh - 1 - t >= 0) {
                let idx = (i + (y + fh - 1 - t) * img.width) * 4;
                img.pixels[idx] = 255; img.pixels[idx+1] = 0; img.pixels[idx+2] = 0;
            }
        }
    }
    for (let j = y; j < y + fh; j++) {
        for (let t = 0; t < thickness; t++) {
            if (x + t < img.width) {
                let idx = ((x + t) + j * img.width) * 4;
                img.pixels[idx] = 255; img.pixels[idx+1] = 0; img.pixels[idx+2] = 0;
            }
            if (x + fw - 1 - t >= 0) {
                let idx = ((x + fw - 1 - t) + j * img.width) * 4;
                img.pixels[idx] = 255; img.pixels[idx+1] = 0; img.pixels[idx+2] = 0;
            }
        }
    }
    img.updatePixels();
}

// ============================================
// THERMAL IMAGING FUNCTIONS
// ============================================
function updateThermalDisplay() {
    if (!currentImage) return;

    applyThermalPixels();
    previousFrame.copy(currentImage, 0, 0, w, h, 0, 0, w, h);

    if (detections.length > 0)                    drawThermalFaceTracking();
    if (currentThermalMode === MODES.MOTION_HEAT) drawHeatTrails();
    drawThermalUIOverlay();

    background(0);
    image(thermalGraphics, 0, 0, THERMAL_WIDTH, THERMAL_HEIGHT);

    updateThermalUI();
}

function applyThermalPixels() {
    switch (currentThermalMode) {
        case MODES.BASIC:          applyBasicThermal();    break;
        case MODES.MOTION_HEAT:    applyMotionThermal();   break;
        case MODES.BODY_HEAT:      applyBodyHeatThermal(); break;
        case MODES.EDGE_DETECTION: applyEdgeDetection();   break;
    }
}

function applyBasicThermal() {
    let buf = createImage(w, h);
    buf.loadPixels();
    currentImage.loadPixels();
    for (let i = 0; i < currentImage.pixels.length; i += 4) {
        const lum    = 0.299 * currentImage.pixels[i] + 0.587 * currentImage.pixels[i+1] + 0.114 * currentImage.pixels[i+2];
        const scaled = constrain(lum * thermalConfig.temperatureScale, 0, 255);
        const col    = getThermalColor(scaled / 255);
        buf.pixels[i] = col[0]; buf.pixels[i+1] = col[1]; buf.pixels[i+2] = col[2]; buf.pixels[i+3] = 255;
    }
    buf.updatePixels();
    thermalGraphics.clear();
    thermalGraphics.image(buf, 0, 0, THERMAL_WIDTH, THERMAL_HEIGHT);
}

function applyMotionThermal() {
    let buf = createImage(w, h);
    buf.loadPixels();
    currentImage.loadPixels();
    previousFrame.loadPixels();
    for (let i = 0; i < currentImage.pixels.length; i += 4) {
        const mapIdx     = i / 4;
        const currBright = (currentImage.pixels[i]  + currentImage.pixels[i+1]  + currentImage.pixels[i+2])  / 3;
        const prevBright = (previousFrame.pixels[i] + previousFrame.pixels[i+1] + previousFrame.pixels[i+2]) / 3;
        const motion     = abs(currBright - prevBright) * thermalConfig.motionSensitivity;
        heatMap[mapIdx]  = constrain(heatMap[mapIdx] + motion, 0, 255) * thermalConfig.coolingRate;
        if (motion > 10) {
            heatTrails.push({
                x: (mapIdx % w) * (THERMAL_WIDTH / w),
                y: floor(mapIdx / w) * (THERMAL_HEIGHT / h),
                life: 255
            });
        }
        const totalHeat = constrain(currBright + heatMap[mapIdx], 0, 255);
        const col = getThermalColor(totalHeat / 255);
        buf.pixels[i] = col[0]; buf.pixels[i+1] = col[1]; buf.pixels[i+2] = col[2]; buf.pixels[i+3] = 255;
    }
    buf.updatePixels();
    thermalGraphics.clear();
    thermalGraphics.image(buf, 0, 0, THERMAL_WIDTH, THERMAL_HEIGHT);
    updateHeatTrails();
}

function applyBodyHeatThermal() {
    let buf = createImage(w, h);
    buf.loadPixels();
    currentImage.loadPixels();
    for (let py = 0; py < h; py++) {
        for (let px = 0; px < w; px++) {
            const idx = (py * w + px) * 4;
            const r = currentImage.pixels[idx], g = currentImage.pixels[idx+1], b = currentImage.pixels[idx+2];
            let heat = 0.299 * r + 0.587 * g + 0.114 * b;
            if (thermalToggles.showFaceHeat) {
                for (let det of detections) {
                    const box = det.alignedRect._box;
                    if (box && px >= box._x && px < box._x + box._width && py >= box._y && py < box._y + box._height) {
                        heat += thermalConfig.faceHeatBoost;
                        if (isSkinTone(r, g, b)) heat += 30;
                        break;
                    }
                }
            }
            heat = constrain(heat, 0, 255);
            const col = getThermalColor(constrain(heat * thermalConfig.temperatureScale, 0, 255) / 255);
            buf.pixels[idx] = col[0]; buf.pixels[idx+1] = col[1]; buf.pixels[idx+2] = col[2]; buf.pixels[idx+3] = 255;
        }
    }
    buf.updatePixels();
    thermalGraphics.clear();
    thermalGraphics.image(buf, 0, 0, THERMAL_WIDTH, THERMAL_HEIGHT);
}

function applyEdgeDetection() {
    let buf = createImage(w, h);
    buf.loadPixels();
    currentImage.loadPixels();
    let edgeCount = 0;
    for (let py = 1; py < h - 1; py++) {
        for (let px = 1; px < w - 1; px++) {
            const idx = (py * w + px) * 4;
            const gx = (getGrayValue(px+1, py-1) + 2*getGrayValue(px+1, py) + getGrayValue(px+1, py+1))
                     - (getGrayValue(px-1, py-1) + 2*getGrayValue(px-1, py) + getGrayValue(px-1, py+1));
            const gy = (getGrayValue(px-1, py+1) + 2*getGrayValue(px, py+1) + getGrayValue(px+1, py+1))
                     - (getGrayValue(px-1, py-1) + 2*getGrayValue(px, py-1) + getGrayValue(px+1, py-1));
            let mag = sqrt(gx*gx + gy*gy) * thermalConfig.edgeStrength;
            if (mag > thermalConfig.edgeThreshold) edgeCount++;
            let r, g, b;
            if (mag > thermalConfig.edgeThreshold * 2) {
                [r, g, b] = EDGE_COLORS.strongEdge;
            } else if (mag > thermalConfig.edgeThreshold) {
                [r, g, b] = EDGE_COLORS.edge;
            } else {
                const bright = getGrayValue(px, py) * 0.2;
                r = constrain(EDGE_COLORS.background[0] + bright, 0, 255);
                g = constrain(EDGE_COLORS.background[1] + bright, 0, 255);
                b = constrain(EDGE_COLORS.background[2] + bright, 0, 255);
            }
            buf.pixels[idx] = r; buf.pixels[idx+1] = g; buf.pixels[idx+2] = b; buf.pixels[idx+3] = 255;
        }
    }
    buf.updatePixels();
    thermalGraphics.clear();
    thermalGraphics.image(buf, 0, 0, THERMAL_WIDTH, THERMAL_HEIGHT);
    thermalStats.edgeCount = edgeCount;
}

function getGrayValue(px, py) {
    const idx = (py * w + px) * 4;
    return 0.299 * currentImage.pixels[idx] + 0.587 * currentImage.pixels[idx+1] + 0.114 * currentImage.pixels[idx+2];
}

function getThermalColor(t) {
    t = constrain(t, 0, 1);
    const index     = floor(t * (THERMAL_PALETTE.length - 1));
    const nextIndex = min(index + 1, THERMAL_PALETTE.length - 1);
    const blend     = t * (THERMAL_PALETTE.length - 1) - index;
    const c1 = THERMAL_PALETTE[index], c2 = THERMAL_PALETTE[nextIndex];
    return [lerp(c1[0], c2[0], blend), lerp(c1[1], c2[1], blend), lerp(c1[2], c2[2], blend)];
}

function isSkinTone(r, g, b) {
    const sum    = r + g + b + 0.01;
    const rRatio = r / sum, gRatio = g / sum;
    return r > 80 && g > 40 && b > 20 && rRatio > 0.35 && rRatio < 0.6 && gRatio > 0.25 && gRatio < 0.45;
}

function updateHeatTrails() {
    for (let i = heatTrails.length - 1; i >= 0; i--) {
        heatTrails[i].life -= 5;
        if (heatTrails[i].life <= 0) heatTrails.splice(i, 1);
    }
}

function drawHeatTrails() {
    for (let trail of heatTrails) {
        thermalGraphics.noStroke();
        thermalGraphics.fill(255, 255, 0, trail.life);
        thermalGraphics.ellipse(trail.x, trail.y, 5, 5);
    }
}

function drawThermalFaceTracking() {
    thermalGraphics.push();
    thermalGraphics.scale(THERMAL_WIDTH / w, THERMAL_HEIGHT / h);
    for (let det of detections) {
        const box = det.alignedRect._box;
        if (thermalToggles.showFaceBoxes) {
            thermalGraphics.noFill(); thermalGraphics.stroke(255, 107, 107); thermalGraphics.strokeWeight(2);
            thermalGraphics.rect(box._x, box._y, box._width, box._height);
        }
        if (thermalToggles.showTemperatureLabels) {
            thermalGraphics.fill(255, 0, 255); thermalGraphics.noStroke(); thermalGraphics.textSize(12); thermalGraphics.textAlign(LEFT);
            thermalGraphics.text(`${thermalStats.faceTemp.toFixed(1)}°C`, box._x, box._y - 5);
        }
        if (thermalToggles.showLandmarks && det.landmarks) {
            thermalGraphics.fill(255, 170, 0); thermalGraphics.noStroke();
            for (let pt of det.landmarks._positions) thermalGraphics.ellipse(pt._x, pt._y, 2, 2);
        }
    }
    thermalGraphics.pop();
}

function drawThermalUIOverlay() {
    thermalGraphics.push();
    thermalGraphics.fill(0, 0, 0, 180); thermalGraphics.noStroke(); thermalGraphics.rect(10, 10, 200, 80, 5);
    thermalGraphics.fill(255, 107, 107); thermalGraphics.textSize(16); thermalGraphics.textAlign(LEFT, TOP);
    const modeNames = ['Basic', 'Motion Heat', 'Body Heat', 'Edge Detection'];
    thermalGraphics.text(`Mode: ${modeNames[currentThermalMode]}`, 20, 20);
    thermalGraphics.textSize(12);
    thermalGraphics.text(`Faces: ${thermalStats.faceCount}`, 20, 50);
    thermalGraphics.text(currentThermalMode === MODES.EDGE_DETECTION ? `Edges: ${thermalStats.edgeCount}` : `Temp Scale: ${thermalConfig.temperatureScale.toFixed(1)}x`, 20, 70);
    drawThermalScaleOnCanvas(THERMAL_WIDTH - 40, 20, 20, THERMAL_HEIGHT - 40);
    thermalGraphics.pop();
}

function drawThermalScaleOnCanvas(x, y, sw, sh) {
    thermalGraphics.push();
    for (let i = 0; i < sh; i++) {
        const col = getThermalColor(i / sh);
        thermalGraphics.stroke(col[0], col[1], col[2]);
        thermalGraphics.line(x, y + i, x + sw, y + i);
    }
    thermalGraphics.noFill(); thermalGraphics.stroke(255); thermalGraphics.strokeWeight(1); thermalGraphics.rect(x, y, sw, sh);
    thermalGraphics.fill(255); thermalGraphics.noStroke(); thermalGraphics.textSize(10); thermalGraphics.textAlign(LEFT, CENTER);
    thermalGraphics.text('Cold', x + sw + 5, y + 10);
    thermalGraphics.text('Hot',  x + sw + 5, y + sh - 10);
    thermalGraphics.pop();
}

function calculateFaceTemperature(detection) {
    const box = detection.alignedRect._box;
    let totalHeat = 0, count = 0;
    const startX = floor(box._x), startY = floor(box._y);
    const endX = min(startX + box._width, w), endY = min(startY + box._height, h);
    currentImage.loadPixels();
    for (let py = startY; py < endY; py += 2) {
        for (let px = startX; px < endX; px += 2) {
            const idx = (py * w + px) * 4;
            if (currentImage.pixels.length > idx + 2) {
                const r = currentImage.pixels[idx], g = currentImage.pixels[idx+1], b = currentImage.pixels[idx+2];
                if (isSkinTone(r, g, b)) {
                    totalHeat += 0.299*r + 0.587*g + 0.114*b;
                    count++;
                }
            }
        }
    }
    return count > 0 ? map(totalHeat / count, 0, 255, 20, 40) : 25;
}

function updateThermalUI() {
    select('#face-count').html(thermalStats.faceCount);
    select('#face-temp').html(thermalStats.faceCount > 0 ? thermalStats.faceTemp.toFixed(1) : '--');
    select('#hot-spots').html(currentThermalMode === MODES.EDGE_DETECTION ? thermalStats.edgeCount : 'N/A');
}

function updateStatus() {
    select('#fps-counter').html(`FPS: ${round(frameRate())}`);
}

function captureSnapshot() {
    if (!currentImage) return;
    snapshot = createImage(w, h);
    snapshot.copy(currentImage, 0, 0, w, h, 0, 0, w, h);
    select('#webcam-status').html('Webcam: Snapshot Active');
}

// ============================================
// RESET FUNCTION
// ============================================
function resetWebcam() {
    snapshot = null;
    faceFilterType = 'none';
    
    detections = [];
    thermalStats.faceCount = 0;
    
    select('#face-status').html('Faces: 0 detected');
    select('#filter-status').html('Filter: None');
    select('#webcam-status').html('Webcam: Resetting...');
    updateButtonStates();
    
    if (capture) {
        capture.remove();
        capture = null;
    }
    
    if (faceapi) {
        faceapi = null;
    }
    
    setTimeout(() => {
        initWebcam();
        setTimeout(() => {
            initFaceDetection();
        }, 500);
    }, 100);
    
    console.log('Webcam reset complete');
}

// ============================================
// KEYBOARD CONTROLS
// ============================================
function keyPressed() {
    if (key === 'M' || key === 'm') { switchToPage(0); return false; }
    if (key === ' ') { switchToPage(currentPage === 0 ? 1 : 0); return false; }

    if (currentPage === 0) {
        if (key === 'x' || key === 'X') { captureSnapshot(); return false; }
        if (key === 'r' || key === 'R') { resetWebcam(); return false; }
        // Keys 1–5 groups toggle on/off
        if (key >= '1' && key <= '5') {
            toggleGroup(parseInt(key) - 1);
            return false;
        }
        if (key === 'g') { setFaceFilter('grayscale'); return false; }
        if (key === 'f') { setFaceFilter('flip');      return false; }
        if (key === 'c') { setFaceFilter('colour');    return false; }
        if (key === 'p') { setFaceFilter('pixelate');  return false; }
    } else if (currentPage === 1) {
        if (key >= '1' && key <= '4') { setThermalMode(parseInt(key) - 1); return false; }
        if (key === 'f' || key === 'F') { toggleThermalFeature('showFaceBoxes',         thermalToggleButtons[0]); return false; }
        if (key === 'l' || key === 'L') { toggleThermalFeature('showLandmarks',          thermalToggleButtons[1]); return false; }
        if (key === 't' || key === 'T') { toggleThermalFeature('showTemperatureLabels',   thermalToggleButtons[2]); return false; }
        if (key === 'h' || key === 'H') { toggleThermalFeature('showFaceHeat',            thermalToggleButtons[3]); return false; }
    }
}