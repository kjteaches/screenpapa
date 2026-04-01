<script>
  // state: image
  let imgEl;
  let imgSrc = "";
  let imgLoaded = false;
  let fileInput;
  let wrapperEl;

  // state: border/frame settings
  let borderSize = 16;
  let borderRadius = 16;
  let windowEnabled = false;

  // state: background mode — 'solid' | 'gradient'
  let bgMode = "solid";

  // solid color
  let borderColor = "f4f5f6";

  // gradient
  const PRESET_GRADIENTS = [
    { id: "g1", name: "Dusk", colors: ["#f9a8d4", "#c084fc", "#818cf8"] },
    { id: "g2", name: "Ocean", colors: ["#67e8f9", "#38bdf8", "#6366f1"] },
    { id: "g3", name: "Peach", colors: ["#fde68a", "#fca5a5", "#f9a8d4"] },
    { id: "g4", name: "Forest", colors: ["#6ee7b7", "#34d399", "#059669"] },
    { id: "g5", name: "Ember", colors: ["#fcd34d", "#fb923c", "#f43f5e"] },
    { id: "g6", name: "Slate", colors: ["#e2e8f0", "#94a3b8", "#475569"] },
  ];
  let selectedGradientId = "g1";
  let customGradientA = "#a855f7";
  let customGradientB = "#3b82f6";
  let useCustomGradient = false;

  // state: copy button feedback
  let copyState = "idle";

  // state: compress
  let compressQuality = 80;
  let showCompressSlider = false;
  let compressState = "idle";

  // state: annotations
  let annotations = [];
  let nextCounterId = 1;
  let selectedId = null;
  let activeTool = null;

  // state: dragging annotations
  let dragging = null;
  let dragOffset = { x: 0, y: 0 };

  // state: drawing rectangles
  let rectStart = null;
  let rectPreview = null;

  // state: crop
  let cropStart = null;
  let cropPreview = null;
  let cropRect = null;
  let cropDragging = null;
  let cropDragStart = null;

  // constants
  const BORDER_SIZES = [0, 4, 8, 16, 24, 32, 48, 64, 128];
  const RADII = [0, 8, 16, 32, 64, 128];

  const CHROME_H = 40;
  const CHROME_BG = "#2a2a2e";
  const CHROME_DOTS = ["#ff5f57", "#febc2e", "#28c840"];
  const CHROME_DOT_R = 6;
  const CHROME_DOT_GAP = 20;
  const CHROME_DOT_LEFT = 16;
  const CHROME_CORNER = 10;

  const PALETTE = ["#ef4444", "#f97316", "#3b82f6", "#22c55e", "#a855f7"];
  let markerColor = "#ef4444";
  let customMarkerColor = "#ff00ff";

  const PIN_R = 16;
  const PIN_TAIL = 24;
  const RECT_CORNER = 8;
  const RECT_STROKE_W = 3;

  $: hexColor = "#" + (borderColor.replace(/^#/, "").trim() || "f4f5f6");
  $: hasImage = !!imgSrc;

  // active gradient colors (preset or custom)
  $: activeGradientColors = useCustomGradient
    ? [customGradientA, customGradientB]
    : (PRESET_GRADIENTS.find((g) => g.id === selectedGradientId)?.colors ??
      PRESET_GRADIENTS[0].colors);

  // CSS mesh gradient string for preview
  function meshGradientCSS(colors) {
    const [c1, c2, c3] =
      colors.length >= 3 ? colors : [colors[0], colors[1], colors[0]];
    return `radial-gradient(ellipse at 0% 0%, ${c1} 0%, transparent 60%),
            radial-gradient(ellipse at 100% 0%, ${c2} 0%, transparent 60%),
            radial-gradient(ellipse at 100% 100%, ${c3 ?? c1} 0%, transparent 60%),
            radial-gradient(ellipse at 0% 100%, ${c2} 0%, transparent 60%),
            ${c1}`;
  }

  function meshGradientCSSCustom(a, b) {
    return `radial-gradient(ellipse at 0% 0%, ${a} 0%, transparent 60%),
            radial-gradient(ellipse at 100% 100%, ${b} 0%, transparent 60%),
            radial-gradient(ellipse at 100% 0%, ${b}88 0%, transparent 50%),
            radial-gradient(ellipse at 0% 100%, ${a}88 0%, transparent 50%),
            ${a}`;
  }

  $: activeMeshCSS = useCustomGradient
    ? meshGradientCSSCustom(customGradientA, customGradientB)
    : meshGradientCSS(activeGradientColors);

  // wrapper background style
  $: wrapperBg = bgMode === "gradient" ? activeMeshCSS : hexColor;

  function makeTeardropPath(r, tail) {
    const c = r * 0.55;
    const ty = tail * 0.45;
    return [
      `M 0 ${-r}`,
      `C ${c} ${-r}, ${r} ${-c}, ${r} 0`,
      `C ${r} ${c}, ${r * 0.7} ${ty}, 0 ${tail}`,
      `C ${-r * 0.7} ${ty}, ${-r} ${c}, ${-r} 0`,
      `C ${-r} ${-c}, ${-c} ${-r}, 0 ${-r}`,
      "Z",
    ].join(" ");
  }

  const teardropPath = makeTeardropPath(PIN_R, PIN_TAIL);

  function getBulbCenter(ann) {
    const angle = ann.rotation - Math.PI / 2;
    return {
      x: ann.tipX - Math.cos(angle) * PIN_TAIL,
      y: ann.tipY - Math.sin(angle) * PIN_TAIL,
    };
  }

  function getImgOffset() {
    return {
      x: borderSize,
      y: borderSize + (windowEnabled ? CHROME_H : 0),
    };
  }

  // IMAGE LOADING

  function loadBlob(blob) {
    if (imgSrc.startsWith("blob:")) URL.revokeObjectURL(imgSrc);
    imgLoaded = false;
    imgSrc = URL.createObjectURL(blob);
    annotations = [];
    nextCounterId = 1;
    selectedId = null;
    activeTool = null;
    cropRect = null;
    cropPreview = null;
    showCompressSlider = false;
  }

  function onImgLoad() {
    imgLoaded = true;
  }

  function handlePaste(e) {
    const items = e.clipboardData?.items;
    if (!items) return;
    for (const item of items) {
      if (item.type.startsWith("image/")) {
        loadBlob(item.getAsFile());
        break;
      }
    }
  }

  function handleFileChange() {
    if (fileInput.files[0]) loadBlob(fileInput.files[0]);
    fileInput.value = "";
  }

  function clearImage() {
    if (imgSrc.startsWith("blob:")) URL.revokeObjectURL(imgSrc);
    imgSrc = "";
    imgLoaded = false;
    annotations = [];
    nextCounterId = 1;
    selectedId = null;
    activeTool = null;
    cropRect = null;
    cropPreview = null;
    showCompressSlider = false;
    compressQuality = 80;
    compressState = "idle";
  }

  // TOOL SELECTION

  function toggleTool(tool) {
    activeTool = activeTool === tool ? null : tool;
    selectedId = null;
    rectStart = null;
    rectPreview = null;
    if (tool !== "crop") {
      cropRect = null;
      cropPreview = null;
      cropStart = null;
    }
  }

  // PLACING ANNOTATIONS

  function onImageAreaClick(e) {
    if (!activeTool || activeTool === "rect" || activeTool === "crop") return;
    if (dragging) return;
    if (e.target.closest(".anno-hit")) return;

    const rect = e.currentTarget.getBoundingClientRect();
    const off = getImgOffset();
    const x = e.clientX - rect.left + off.x;
    const y = e.clientY - rect.top + off.y;

    const id = Date.now() + Math.random();
    const ann = {
      id,
      type: activeTool,
      color: markerColor,
      tipX: x,
      tipY: y,
      rotation: 0,
    };
    if (activeTool === "counter") ann.count = nextCounterId++;

    annotations = [...annotations, ann];
    selectedId = id;
    if (activeTool === "arrow") activeTool = null;
  }

  // DRAWING RECTANGLES

  function onImageAreaMouseDown(e) {
    if (activeTool === "rect") {
      if (e.target.closest(".anno-hit")) return;
      e.preventDefault();
      const rect = e.currentTarget.getBoundingClientRect();
      const off = getImgOffset();
      rectStart = {
        x: e.clientX - rect.left + off.x,
        y: e.clientY - rect.top + off.y,
      };
      rectPreview = null;
      return;
    }
    if (activeTool === "crop") {
      if (e.target.closest(".crop-handle") || e.target.closest(".crop-area"))
        return;
      e.preventDefault();
      const rect = e.currentTarget.getBoundingClientRect();
      cropStart = { x: e.clientX - rect.left, y: e.clientY - rect.top };
      cropPreview = null;
      cropRect = null;
      return;
    }
  }

  function onImageAreaMouseMove(e) {
    if (activeTool === "rect" && rectStart) {
      const rect = e.currentTarget.getBoundingClientRect();
      const off = getImgOffset();
      const cx = e.clientX - rect.left + off.x;
      const cy = e.clientY - rect.top + off.y;
      rectPreview = {
        x: Math.min(rectStart.x, cx),
        y: Math.min(rectStart.y, cy),
        w: Math.abs(cx - rectStart.x),
        h: Math.abs(cy - rectStart.y),
      };
      return;
    }
    if (activeTool === "crop" && cropStart) {
      const rect = e.currentTarget.getBoundingClientRect();
      const cx = e.clientX - rect.left;
      const cy = e.clientY - rect.top;
      const imgW = imgEl.getBoundingClientRect().width;
      const imgH = imgEl.getBoundingClientRect().height;
      cropPreview = {
        x: Math.max(0, Math.min(cropStart.x, cx)),
        y: Math.max(0, Math.min(cropStart.y, cy)),
        w: Math.min(Math.abs(cx - cropStart.x), imgW),
        h: Math.min(Math.abs(cy - cropStart.y), imgH),
      };
      if (cropPreview.x + cropPreview.w > imgW)
        cropPreview.w = imgW - cropPreview.x;
      if (cropPreview.y + cropPreview.h > imgH)
        cropPreview.h = imgH - cropPreview.y;
      return;
    }
  }

  function onImageAreaMouseUp() {
    if (activeTool === "rect" && rectStart) {
      if (rectPreview && rectPreview.w > 5 && rectPreview.h > 5) {
        const id = Date.now() + Math.random();
        annotations = [
          ...annotations,
          {
            id,
            type: "rect",
            color: markerColor,
            rx: rectPreview.x,
            ry: rectPreview.y,
            rw: rectPreview.w,
            rh: rectPreview.h,
          },
        ];
        selectedId = id;
      }
      rectStart = null;
      rectPreview = null;
      return;
    }
    if (activeTool === "crop" && cropStart) {
      cropRect =
        cropPreview && cropPreview.w > 10 && cropPreview.h > 10
          ? { ...cropPreview }
          : null;
      cropStart = null;
      cropPreview = null;
      return;
    }
  }

  // CROP HANDLE DRAGGING

  function getImgRelPos(e) {
    const rect = imgEl.getBoundingClientRect();
    return {
      x: Math.max(0, Math.min(e.clientX - rect.left, rect.width)),
      y: Math.max(0, Math.min(e.clientY - rect.top, rect.height)),
    };
  }

  function onCropHandleDown(e, handle) {
    e.stopPropagation();
    e.preventDefault();
    cropDragging = handle;
    cropDragStart = getImgRelPos(e);
  }
  function onCropAreaDown(e) {
    e.stopPropagation();
    e.preventDefault();
    cropDragging = "move";
    cropDragStart = getImgRelPos(e);
  }

  function onGlobalPointerMove(e) {
    if (dragging) {
      const pos = getRelPos(e);
      annotations = annotations.map((a) => {
        if (a.id !== dragging.id) return a;
        if (a.type === "rect")
          return { ...a, rx: pos.x - dragOffset.x, ry: pos.y - dragOffset.y };
        if (dragging.part === "tip") {
          const bulb = getBulbCenter(a);
          const angle = Math.atan2(pos.y - bulb.y, pos.x - bulb.x);
          return {
            ...a,
            tipX: bulb.x + Math.cos(angle) * PIN_TAIL,
            tipY: bulb.y + Math.sin(angle) * PIN_TAIL,
            rotation: angle + Math.PI / 2,
          };
        }
        const newBulbX = pos.x - dragOffset.x;
        const newBulbY = pos.y - dragOffset.y;
        const angle = a.rotation - Math.PI / 2;
        return {
          ...a,
          tipX: newBulbX + Math.cos(angle) * PIN_TAIL,
          tipY: newBulbY + Math.sin(angle) * PIN_TAIL,
        };
      });
      return;
    }
    if (cropDragging && cropRect) {
      const pos = getImgRelPos(e);
      const dx = pos.x - cropDragStart.x;
      const dy = pos.y - cropDragStart.y;
      const imgW = imgEl.getBoundingClientRect().width;
      const imgH = imgEl.getBoundingClientRect().height;
      let { x, y, w, h } = cropRect;
      if (cropDragging === "move") {
        x = Math.max(0, Math.min(x + dx, imgW - w));
        y = Math.max(0, Math.min(y + dy, imgH - h));
      } else if (cropDragging === "nw") {
        const nx = Math.max(0, x + dx),
          ny = Math.max(0, y + dy);
        w = w + (x - nx);
        h = h + (y - ny);
        x = nx;
        y = ny;
      } else if (cropDragging === "ne") {
        const ny = Math.max(0, y + dy);
        w = Math.min(w + dx, imgW - x);
        h = h + (y - ny);
        y = ny;
      } else if (cropDragging === "sw") {
        const nx = Math.max(0, x + dx);
        w = w + (x - nx);
        h = Math.min(h + dy, imgH - y);
        x = nx;
      } else if (cropDragging === "se") {
        w = Math.min(w + dx, imgW - x);
        h = Math.min(h + dy, imgH - y);
      } else if (cropDragging === "n") {
        const ny = Math.max(0, y + dy);
        h = h + (y - ny);
        y = ny;
      } else if (cropDragging === "s") {
        h = Math.min(h + dy, imgH - y);
      } else if (cropDragging === "w") {
        const nx = Math.max(0, x + dx);
        w = w + (x - nx);
        x = nx;
      } else if (cropDragging === "e") {
        w = Math.min(w + dx, imgW - x);
      }
      if (w > 10 && h > 10) cropRect = { x, y, w, h };
      cropDragStart = pos;
    }
  }

  function onGlobalPointerUp() {
    dragging = null;
    cropDragging = null;
    cropDragStart = null;
  }

  // APPLY CROP

  function applyCrop() {
    if (!cropRect || !imgEl || !imgLoaded) return;
    const dispW = imgEl.getBoundingClientRect().width;
    const dispH = imgEl.getBoundingClientRect().height;
    const sx = imgEl.naturalWidth / dispW;
    const sy = imgEl.naturalHeight / dispH;
    const cx = Math.round(cropRect.x * sx),
      cy = Math.round(cropRect.y * sy);
    const cw = Math.round(cropRect.w * sx),
      ch = Math.round(cropRect.h * sy);
    const canvas = document.createElement("canvas");
    canvas.width = cw;
    canvas.height = ch;
    canvas.getContext("2d").drawImage(imgEl, cx, cy, cw, ch, 0, 0, cw, ch);
    canvas.toBlob((blob) => {
      if (!blob) return;
      if (imgSrc.startsWith("blob:")) URL.revokeObjectURL(imgSrc);
      imgLoaded = false;
      imgSrc = URL.createObjectURL(blob);
      cropRect = null;
      cropPreview = null;
      activeTool = null;
      annotations = [];
      nextCounterId = 1;
      selectedId = null;
    }, "image/png");
  }

  function cancelCrop() {
    cropRect = null;
    cropPreview = null;
    cropStart = null;
    activeTool = null;
  }

  // COMPRESS

  function toggleCompressSlider() {
    showCompressSlider = !showCompressSlider;
  }

  function compressImage() {
    if (!imgEl || !imgLoaded) return;
    const canvas = document.createElement("canvas");
    canvas.width = imgEl.naturalWidth;
    canvas.height = imgEl.naturalHeight;
    canvas.getContext("2d").drawImage(imgEl, 0, 0);
    canvas.toBlob(
      (blob) => {
        if (!blob) {
          compressState = "error";
          setTimeout(() => (compressState = "idle"), 1500);
          return;
        }
        if (imgSrc.startsWith("blob:")) URL.revokeObjectURL(imgSrc);
        imgLoaded = false;
        imgSrc = URL.createObjectURL(blob);
        compressState = "success";
        setTimeout(() => (compressState = "idle"), 1500);
      },
      "image/jpeg",
      compressQuality / 100,
    );
  }

  // ANNOTATION DRAGGING

  function getRelPos(e) {
    const rect = wrapperEl.getBoundingClientRect();
    return { x: e.clientX - rect.left, y: e.clientY - rect.top };
  }

  function onAnnotationPointerDown(e, ann, part) {
    e.stopPropagation();
    e.preventDefault();
    selectedId = ann.id;
    const pos = getRelPos(e);
    if (ann.type === "rect") {
      dragOffset = { x: pos.x - ann.rx, y: pos.y - ann.ry };
    } else if (part === "bulb") {
      const bulb = getBulbCenter(ann);
      dragOffset = { x: pos.x - bulb.x, y: pos.y - bulb.y };
    }
    dragging = { id: ann.id, part };
  }

  function deleteSelected() {
    if (selectedId == null) return;
    annotations = annotations.filter((a) => a.id !== selectedId);
    selectedId = null;
  }

  // KEYBOARD

  function handleKeydown(e) {
    if (e.key === "Escape") {
      selectedId = null;
      activeTool = null;
      rectStart = null;
      rectPreview = null;
      cropRect = null;
      cropPreview = null;
      cropStart = null;
      return;
    }
    const isTyping =
      document.activeElement?.tagName === "INPUT" ||
      document.activeElement?.tagName === "SELECT";
    if (
      (e.key === "Delete" || e.key === "Backspace") &&
      selectedId != null &&
      !isTyping
    ) {
      e.preventDefault();
      deleteSelected();
    }
    if (e.key === "Enter" && activeTool === "crop" && cropRect) {
      e.preventDefault();
      applyCrop();
    }
  }

  // CLICK-AWAY

  function onAppClick(e) {
    const isCanvasArea = e.target.closest(".canvas-area");
    const isToolControl =
      e.target.closest(".anno-buttons") ||
      e.target.closest(".marker-colors") ||
      e.target.closest(".custom-color-wrap") ||
      e.target.closest(".delete-btn") ||
      e.target.closest(".crop-actions") ||
      e.target.closest(".compress-section") ||
      e.target.closest(".bg-tabs");
    if (!isCanvasArea && !isToolControl) {
      activeTool = null;
      rectStart = null;
      rectPreview = null;
      cropRect = null;
      cropPreview = null;
    }
  }

  function onWrapperClick(e) {
    if (activeTool) return;
    if (e.target === wrapperEl || e.target === imgEl) selectedId = null;
  }

  function onCanvasBgClick(e) {
    if (e.target.closest(".wrapper")) return;
    if (activeTool === "crop") return;
    activeTool = null;
    rectStart = null;
    rectPreview = null;
    selectedId = null;
  }

  // EXPORT — draw gradient or solid onto canvas

  function drawBgToCanvas(ctx, w, h) {
    if (bgMode === "gradient") {
      const colors = useCustomGradient
        ? [customGradientA, customGradientB]
        : (PRESET_GRADIENTS.find((g) => g.id === selectedGradientId)?.colors ??
          PRESET_GRADIENTS[0].colors);
      const [c1, c2, c3] =
        colors.length >= 3 ? colors : [colors[0], colors[1], colors[0]];

      // base fill
      ctx.fillStyle = c1;
      ctx.fillRect(0, 0, w, h);

      // top-left blob
      const g1 = ctx.createRadialGradient(0, 0, 0, 0, 0, w * 0.8);
      g1.addColorStop(0, c1 + "ff");
      g1.addColorStop(1, c1 + "00");
      ctx.fillStyle = g1;
      ctx.fillRect(0, 0, w, h);

      // top-right blob
      const g2 = ctx.createRadialGradient(w, 0, 0, w, 0, w * 0.8);
      g2.addColorStop(0, c2 + "ff");
      g2.addColorStop(1, c2 + "00");
      ctx.fillStyle = g2;
      ctx.fillRect(0, 0, w, h);

      // bottom-right blob
      const c3hex = c3 ?? c1;
      const g3 = ctx.createRadialGradient(w, h, 0, w, h, w * 0.8);
      g3.addColorStop(0, c3hex + "ff");
      g3.addColorStop(1, c3hex + "00");
      ctx.fillStyle = g3;
      ctx.fillRect(0, 0, w, h);

      // bottom-left blob
      const g4 = ctx.createRadialGradient(0, h, 0, 0, h, w * 0.8);
      g4.addColorStop(0, c2 + "cc");
      g4.addColorStop(1, c2 + "00");
      ctx.fillStyle = g4;
      ctx.fillRect(0, 0, w, h);
    } else {
      ctx.fillStyle = hexColor;
      ctx.fillRect(0, 0, w, h);
    }
  }

  function renderCanvas() {
    if (!imgEl || !imgLoaded) return null;
    const natW = imgEl.naturalWidth;
    const natH = imgEl.naturalHeight;
    const dispW = imgEl.getBoundingClientRect().width;
    const dispH = imgEl.getBoundingClientRect().height;
    if (!natW || !natH || !dispW || !dispH) return null;

    const sx = natW / dispW;
    const sy = natH / dispH;
    const pad = Math.round(borderSize * sx);
    const rad = Math.round(borderRadius * sx);
    const chromeH = windowEnabled ? Math.round(CHROME_H * sx) : 0;

    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    canvas.width = natW + pad * 2;
    canvas.height = natH + pad * 2 + chromeH;

    drawBgToCanvas(ctx, canvas.width, canvas.height);

    const imgX = pad;
    const imgY = pad + chromeH;

    if (windowEnabled) {
      const r = Math.round(CHROME_CORNER * sx);
      ctx.fillStyle = CHROME_BG;
      ctx.beginPath();
      ctx.moveTo(pad + r, pad);
      ctx.arcTo(pad + natW, pad, pad + natW, pad + chromeH, r);
      ctx.arcTo(pad + natW, pad + chromeH, pad, pad + chromeH, 0);
      ctx.lineTo(pad, pad + chromeH);
      ctx.arcTo(pad, pad, pad + natW, pad, r);
      ctx.closePath();
      ctx.fill();
      const dr = Math.round(CHROME_DOT_R * sx);
      const dg = Math.round(CHROME_DOT_GAP * sx);
      const dl = Math.round(CHROME_DOT_LEFT * sx);
      for (let i = 0; i < 3; i++) {
        ctx.fillStyle = CHROME_DOTS[i];
        ctx.beginPath();
        ctx.arc(pad + dl + dr + i * dg, pad + chromeH / 2, dr, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    if (!windowEnabled && rad > 0) {
      ctx.save();
      ctx.beginPath();
      ctx.moveTo(imgX + rad, imgY);
      ctx.arcTo(imgX + natW, imgY, imgX + natW, imgY + natH, rad);
      ctx.arcTo(imgX + natW, imgY + natH, imgX, imgY + natH, rad);
      ctx.arcTo(imgX, imgY + natH, imgX, imgY, rad);
      ctx.arcTo(imgX, imgY, imgX + natW, imgY, rad);
      ctx.closePath();
      ctx.clip();
      ctx.drawImage(imgEl, imgX, imgY);
      ctx.restore();
    } else {
      ctx.drawImage(imgEl, imgX, imgY);
    }

    const pox = borderSize;
    const poy = borderSize + (windowEnabled ? CHROME_H : 0);
    for (const a of annotations) {
      if (a.type === "rect")
        drawRectOnCanvas(ctx, a, sx, sy, imgX, imgY, pox, poy);
      else drawPinOnCanvas(ctx, a, sx, sy, imgX, imgY, pox, poy);
    }
    return canvas;
  }

  function drawRectOnCanvas(ctx, a, sx, sy, imgX, imgY, pox, poy) {
    const x = (a.rx - pox) * sx + imgX,
      y = (a.ry - poy) * sy + imgY;
    const w = a.rw * sx,
      h = a.rh * sy,
      r = RECT_CORNER * sx;
    ctx.strokeStyle = a.color;
    ctx.lineWidth = RECT_STROKE_W * sx;
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.arcTo(x + w, y, x + w, y + h, r);
    ctx.arcTo(x + w, y + h, x, y + h, r);
    ctx.arcTo(x, y + h, x, y, r);
    ctx.arcTo(x, y, x + w, y, r);
    ctx.closePath();
    ctx.stroke();
  }

  function drawPinOnCanvas(ctx, a, sx, sy, imgX, imgY, pox, poy) {
    const tipX = (a.tipX - pox) * sx + imgX,
      tipY = (a.tipY - poy) * sy + imgY;
    const tail = PIN_TAIL * sx,
      r = PIN_R * sx;
    const angle = a.rotation - Math.PI / 2;
    const bx = tipX - Math.cos(angle) * tail,
      by = tipY - Math.sin(angle) * tail;
    ctx.save();
    ctx.translate(bx, by);
    ctx.rotate(a.rotation);
    const c = r * 0.55,
      ty = tail * 0.45;
    ctx.beginPath();
    ctx.moveTo(0, -r);
    ctx.bezierCurveTo(c, -r, r, -c, r, 0);
    ctx.bezierCurveTo(r, c, r * 0.7, ty, 0, tail);
    ctx.bezierCurveTo(-r * 0.7, ty, -r, c, -r, 0);
    ctx.bezierCurveTo(-r, -c, -c, -r, 0, -r);
    ctx.closePath();
    ctx.fillStyle = a.color;
    ctx.fill();
    ctx.strokeStyle = "rgba(0,0,0,0.15)";
    ctx.lineWidth = 1.2 * sx;
    ctx.stroke();
    ctx.restore();
    if (a.type === "counter") {
      ctx.save();
      ctx.translate(bx, by);
      ctx.fillStyle = "white";
      ctx.font = `bold ${Math.round(r * 1.15)}px system-ui, sans-serif`;
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillText(String(a.count), 0, -1 * sx);
      ctx.restore();
    }
  }

  function copyImage() {
    if (!hasImage || !imgLoaded) return;
    const canvas = renderCanvas();
    if (!canvas) return;
    canvas.toBlob((blob) => {
      if (!blob) {
        copyState = "error";
        return;
      }
      navigator.clipboard
        .write([new ClipboardItem({ "image/png": blob })])
        .then(() => {
          copyState = "success";
        })
        .catch(() => {
          copyState = "error";
        });
      setTimeout(() => (copyState = "idle"), 1500);
    }, "image/png");
  }

  function downloadImage() {
    if (!hasImage || !imgLoaded) return;
    const canvas = renderCanvas();
    if (!canvas) return;
    canvas.toBlob((blob) => {
      if (!blob) return;
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "screenshot.png";
      a.style.display = "none";
      document.body.appendChild(a);
      a.click();
      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 100);
    }, "image/png");
  }

  $: displayCrop = cropRect || cropPreview;
</script>

<svelte:window
  on:paste={handlePaste}
  on:keydown={handleKeydown}
  on:pointermove={onGlobalPointerMove}
  on:pointerup={onGlobalPointerUp}
/>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div class="app" on:click={onAppClick}>
  <div class="sidebar">
    <!-- BORDER SECTION -->
    <div class="section-label">Border</div>

    <!-- SIZE -->
    <div class="field field-inline">
      <span class="label">Size</span>
      <div class="select-wrap select-wrap-sm">
        <select bind:value={borderSize}>
          {#each BORDER_SIZES as s}
            <option value={s}>{s === 0 ? "None" : s + "px"}</option>
          {/each}
        </select>
        <svg
          class="chevron"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M19 9l-7 7-7-7"
          />
        </svg>
      </div>
    </div>

    <!-- RADIUS -->
    <div class="field field-inline" class:disabled-field={windowEnabled}>
      <span class="label">Radius</span>
      <div class="select-wrap select-wrap-sm">
        <select bind:value={borderRadius} disabled={windowEnabled}>
          {#each RADII as r}
            <option value={r}>{r === 0 ? "None" : r + "px"}</option>
          {/each}
        </select>
        <svg
          class="chevron"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M19 9l-7 7-7-7"
          />
        </svg>
      </div>
    </div>

    <!-- WINDOW -->
    <div class="field field-inline">
      <span class="label">Window</span>
      <button
        class="toggle"
        class:on={windowEnabled}
        on:click={() => (windowEnabled = !windowEnabled)}
        role="switch"
        aria-checked={windowEnabled}
      >
        <span class="toggle-dot" class:on={windowEnabled}></span>
      </button>
    </div>

    <!-- BG TABS -->
    <div class="bg-tabs field">
      <div class="tab-row">
        <button
          class="tab-btn"
          class:tab-active={bgMode === "solid"}
          on:click={() => (bgMode = "solid")}>Solid</button
        >
        <button
          class="tab-btn"
          class:tab-active={bgMode === "gradient"}
          on:click={() => (bgMode = "gradient")}>Gradient</button
        >
      </div>

      {#if bgMode === "solid"}
        <!-- SOLID COLOR PANEL -->
        <div class="solid-panel">
          <div class="solid-swatches">
            <!-- Light -->
            <button
              class="color-swatch"
              class:active={borderColor.toLowerCase() === "f4f5f6"}
              style="background:#f4f5f6; border:1px solid #3f3f46"
              on:click={() => (borderColor = "f4f5f6")}
            ></button>
            <!-- Dark -->
            <button
              class="color-swatch"
              class:active={borderColor.toLowerCase() === "18181b"}
              style="background:#18181b; border:1px solid #3f3f46"
              on:click={() => (borderColor = "18181b")}
            ></button>
            <!-- Warm -->
            <button
              class="color-swatch"
              class:active={"#" + borderColor.replace(/^#/, "") === "#f97316"}
              style="background:#f97316"
              on:click={() => (borderColor = "f97316")}
            ></button>
            <!-- Cool -->
            <button
              class="color-swatch"
              class:active={"#" + borderColor.replace(/^#/, "") === "#3b82f6"}
              style="background:#3b82f6"
              on:click={() => (borderColor = "3b82f6")}
            ></button>
            <!-- + custom color picker -->
            <div class="plus-color-wrap" title="Custom color">
              <input
                type="color"
                value={hexColor}
                class="plus-color-input"
                on:input={(e) =>
                  (borderColor = e.target.value.replace("#", ""))}
              />
              <svg
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2.5"
                stroke-linecap="round"
                class="plus-icon"
              >
                <path d="M12 5v14M5 12h14" />
              </svg>
            </div>
          </div>
        </div>
      {:else}
        <!-- GRADIENT PANEL -->
        <div class="gradient-panel">
          <!-- preset swatches -->
          <div class="gradient-swatches">
            {#each PRESET_GRADIENTS as g}
              <button
                class="grad-swatch"
                class:active={selectedGradientId === g.id && !useCustomGradient}
                style="background: {meshGradientCSS(g.colors)}"
                title={g.name}
                on:click={() => {
                  selectedGradientId = g.id;
                  useCustomGradient = false;
                }}
              ></button>
            {/each}
          </div>

          <!-- custom gradient -->
          <div class="custom-grad-section">
            <button
              class="custom-grad-toggle"
              class:active={useCustomGradient}
              on:click={() => (useCustomGradient = true)}
            >
              Custom
            </button>

            {#if useCustomGradient}
              <div class="custom-grad-pickers">
                <div class="custom-grad-picker-wrap" title="Color A">
                  <input
                    type="color"
                    bind:value={customGradientA}
                    class="grad-color-input"
                  />
                  <div
                    class="grad-color-preview"
                    style="background:{customGradientA}"
                  ></div>
                </div>
                <div class="custom-grad-arrow">→</div>
                <div class="custom-grad-picker-wrap" title="Color B">
                  <input
                    type="color"
                    bind:value={customGradientB}
                    class="grad-color-input"
                  />
                  <div
                    class="grad-color-preview"
                    style="background:{customGradientB}"
                  ></div>
                </div>
              </div>
              <!-- live preview -->
              <div
                class="custom-grad-preview"
                style="background:{meshGradientCSSCustom(
                  customGradientA,
                  customGradientB,
                )}"
              ></div>
            {/if}
          </div>
        </div>
      {/if}
    </div>

    <!-- ANNOTATIONS SECTION -->
    <div class="section-label">Annotations</div>

    <!-- MARKER COLORS -->
    <div class="marker-colors">
      {#each PALETTE as pc}
        <button
          class="color-swatch"
          class:active={markerColor === pc}
          style="background:{pc}"
          on:click={() => (markerColor = pc)}
        ></button>
      {/each}
      <!-- + custom marker color picker -->
      <div class="plus-color-wrap" title="Custom color">
        <input
          type="color"
          bind:value={customMarkerColor}
          class="plus-color-input"
          on:input={() => (markerColor = customMarkerColor)}
        />
        <svg
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2.5"
          stroke-linecap="round"
          class="plus-icon"
        >
          <path d="M12 5v14M5 12h14" />
        </svg>
      </div>
    </div>

    <!-- ANNO TOOLS -->
    <div class="anno-buttons">
      <button
        class="tool-btn"
        class:enabled={hasImage}
        class:tool-active={activeTool === "arrow"}
        disabled={!hasImage}
        on:click|stopPropagation={() => toggleTool("arrow")}
        title="Arrow"
      >
        <svg
          width="18"
          height="18"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2.5"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><path d="M12 5V19" /><path d="M5 12l7-7 7 7" /></svg
        >
      </button>
      <button
        class="tool-btn"
        class:enabled={hasImage}
        class:tool-active={activeTool === "counter"}
        disabled={!hasImage}
        on:click|stopPropagation={() => toggleTool("counter")}
        title="Counter"
      >
        <svg
          width="18"
          height="18"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><circle cx="12" cy="12" r="9" /><text
            x="12"
            y="12.5"
            text-anchor="middle"
            dominant-baseline="middle"
            fill="currentColor"
            font-size="11"
            font-weight="bold"
            stroke="none">2</text
          ></svg
        >
      </button>
      <button
        class="tool-btn"
        class:enabled={hasImage}
        class:tool-active={activeTool === "rect"}
        disabled={!hasImage}
        on:click|stopPropagation={() => toggleTool("rect")}
        title="Rectangle"
      >
        <svg
          width="18"
          height="18"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><rect x="3" y="5" width="18" height="14" rx="3" /></svg
        >
      </button>
      <button
        class="tool-btn"
        class:enabled={hasImage}
        class:tool-active={activeTool === "crop"}
        disabled={!hasImage}
        on:click|stopPropagation={() => toggleTool("crop")}
        title="Crop"
      >
        <svg
          width="18"
          height="18"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><path d="M6 2v4H2" /><path d="M18 22v-4h4" /><rect
            x="6"
            y="6"
            width="12"
            height="12"
          /></svg
        >
      </button>
    </div>

    {#if activeTool === "crop" && cropRect}
      <div class="crop-actions">
        <button class="btn crop-apply" on:click|stopPropagation={applyCrop}>
          <svg
            class="icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2.5"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M5 13l4 4L19 7"
            /></svg
          >
          <span>Apply</span>
        </button>
        <button
          class="btn crop-cancel enabled"
          on:click|stopPropagation={cancelCrop}
        >
          <svg
            class="icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M6 18L18 6M6 6l12 12"
            /></svg
          >
          <span>Cancel</span>
        </button>
      </div>
    {/if}

    {#if selectedId != null}
      <button
        class="btn delete-btn enabled"
        on:click|stopPropagation={deleteSelected}
      >
        <svg
          class="icon"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          ><path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
          /></svg
        >
        <span>Delete</span>
      </button>
    {/if}

    <!-- IMAGE SECTION -->
    <div class="section-label">Image</div>

    <!-- COMPRESS (inline panel, toggled from actions row) -->
    {#if showCompressSlider}
      <div class="compress-panel">
        <div class="compress-quality-row">
          <span class="compress-label">Quality</span>
          <span class="compress-value">{compressQuality}%</span>
        </div>
        <input
          type="range"
          min="10"
          max="100"
          step="5"
          bind:value={compressQuality}
          class="quality-slider"
        />
        <div class="compress-hint">
          {#if compressQuality >= 85}Highest quality · larger file{:else if compressQuality >= 50}Balanced
            quality &amp; size{:else}Smallest file · visible artifacts{/if}
        </div>
        <button
          class="btn compress-apply-btn enabled"
          class:success={compressState === "success"}
          class:error={compressState === "error"}
          on:click|stopPropagation={compressImage}
        >
          {#if compressState === "success"}<svg
              class="icon"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2.5"
              ><path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M5 13l4 4L19 7"
              /></svg
            ><span>Done</span>
          {:else if compressState === "error"}<span>Failed</span>
          {:else}<svg
              class="icon"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              ><polyline points="8 17 12 21 16 17" /><line
                x1="12"
                y1="21"
                x2="12"
                y2="3"
              /></svg
            ><span>Apply</span>{/if}
        </button>
      </div>
    {/if}

    <!-- ACTIONS -->
    <div class="actions">
      <button
        class="btn copy-btn"
        class:enabled={hasImage && imgLoaded}
        class:success={copyState === "success"}
        class:error={copyState === "error"}
        disabled={!hasImage || !imgLoaded}
        on:click={copyImage}
      >
        {#if copyState === "success"}<svg
            class="icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M5 13l4 4L19 7"
            /></svg
          ><span>Copied</span>
        {:else if copyState === "error"}<span>Failed</span>
        {:else}<svg
            class="icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            ><rect x="9" y="9" width="13" height="13" rx="2" /><path
              d="M5 15H4a2 2 0 01-2-2V4a2 2 0 012-2h9a2 2 0 012 2v1"
            /></svg
          ><span>Copy</span>{/if}
      </button>
      <button
        class="btn"
        class:enabled={hasImage && imgLoaded}
        disabled={!hasImage || !imgLoaded}
        on:click={downloadImage}
      >
        <svg
          class="icon"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          ><path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M4 16v2a2 2 0 002 2h12a2 2 0 002-2v-2M12 4v12M7 15l5 5 5-5"
          /></svg
        >
        <span>Download</span>
      </button>
      <!-- icon-only row: compress toggle + clear -->
      <div class="actions-secondary">
        <button
          class="btn icon-btn"
          class:enabled={hasImage && imgLoaded}
          class:icon-btn-active={showCompressSlider}
          disabled={!hasImage || !imgLoaded}
          on:click|stopPropagation={toggleCompressSlider}
          title="Compress"
        >
          <svg
            class="icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <line x1="5" y1="12" x2="19" y2="12" />
            <path d="M12 4v5M9 6l3 3 3-3" />
            <path d="M12 20v-5M9 18l3-3 3 3" />
          </svg>
          <span>Compress</span>
        </button>
        <button
          class="btn icon-btn"
          class:enabled={hasImage}
          disabled={!hasImage}
          on:click={clearImage}
          title="Clear"
        >
          <svg
            class="icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M6 18L18 6M6 6l12 12"
            /></svg
          >
          <span>Clear</span>
        </button>
      </div>
    </div>

    <input
      bind:this={fileInput}
      type="file"
      accept="image/*"
      on:change={handleFileChange}
      class="hidden-input"
    />
  </div>

  <div class="vdivider"></div>

  <div class="main-area">
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="canvas-area" on:click={onCanvasBgClick}>
      {#if !hasImage}
        <div class="prompt">
          <div class="prompt-title">Paste an image</div>
          <div class="prompt-sub">Press <kbd>⌘V</kbd> anywhere</div>
          <div class="prompt-or">or</div>
          <button class="upload-btn" on:click={() => fileInput.click()}>
            <svg
              class="icon"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              ><path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M4 16v2a2 2 0 002 2h12a2 2 0 002-2v-2M12 4v12M7 9l5-5 5 5"
              /></svg
            >
            <span>Upload</span>
          </button>
        </div>
      {:else}
        <div class="output" on:click|stopPropagation>
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <div
            bind:this={wrapperEl}
            class="wrapper"
            class:tool-cursor={activeTool != null}
            style="padding:{borderSize}px; background:{wrapperBg}; position:relative"
            on:click={onWrapperClick}
          >
            {#if windowEnabled}
              <div class="window-chrome">
                <div class="wdot red"></div>
                <div class="wdot yellow"></div>
                <div class="wdot green"></div>
              </div>
            {/if}

            <!-- svelte-ignore a11y-no-static-element-interactions -->
            <div
              style="position:relative"
              on:click={onImageAreaClick}
              on:mousedown={onImageAreaMouseDown}
              on:mousemove={onImageAreaMouseMove}
              on:mouseup={onImageAreaMouseUp}
            >
              <img
                bind:this={imgEl}
                src={imgSrc}
                alt="Pasted"
                class="preview-img"
                class:tool-cursor={activeTool != null}
                style="border-radius:{windowEnabled
                  ? '0'
                  : borderRadius + 'px'}"
                on:load={onImgLoad}
              />

              {#if activeTool === "crop" && displayCrop}
                {@const c = displayCrop}
                <div class="crop-overlay">
                  <div
                    class="crop-shade crop-shade-top"
                    style="height:{c.y}px"
                  ></div>
                  <div
                    class="crop-shade crop-shade-bottom"
                    style="top:{c.y + c.h}px; bottom:0"
                  ></div>
                  <div
                    class="crop-shade crop-shade-left"
                    style="top:{c.y}px; height:{c.h}px; width:{c.x}px"
                  ></div>
                  <div
                    class="crop-shade crop-shade-right"
                    style="top:{c.y}px; height:{c.h}px; left:{c.x +
                      c.w}px; right:0"
                  ></div>
                  <!-- svelte-ignore a11y-no-static-element-interactions -->
                  <div
                    class="crop-area"
                    style="left:{c.x}px; top:{c.y}px; width:{c.w}px; height:{c.h}px"
                    on:pointerdown={onCropAreaDown}
                  >
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle nw"
                      on:pointerdown={(e) => onCropHandleDown(e, "nw")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle ne"
                      on:pointerdown={(e) => onCropHandleDown(e, "ne")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle sw"
                      on:pointerdown={(e) => onCropHandleDown(e, "sw")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle se"
                      on:pointerdown={(e) => onCropHandleDown(e, "se")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle n"
                      on:pointerdown={(e) => onCropHandleDown(e, "n")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle s"
                      on:pointerdown={(e) => onCropHandleDown(e, "s")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle w"
                      on:pointerdown={(e) => onCropHandleDown(e, "w")}
                    ></div>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <div
                      class="crop-handle e"
                      on:pointerdown={(e) => onCropHandleDown(e, "e")}
                    ></div>
                  </div>
                </div>
              {/if}

              <svg class="annotation-svg">
                <defs>
                  <filter
                    id="pin-shadow"
                    x="-50%"
                    y="-50%"
                    width="200%"
                    height="200%"
                  >
                    <feDropShadow
                      dx="0"
                      dy="1.5"
                      stdDeviation="2"
                      flood-color="rgba(0,0,0,0.35)"
                    />
                  </filter>
                </defs>
                {#each annotations as a (a.id)}
                  {@const off = {
                    x: -borderSize,
                    y: -(borderSize + (windowEnabled ? CHROME_H : 0)),
                  }}
                  {@const isSelected = selectedId === a.id}
                  {#if a.type === "rect"}
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <rect
                      class="anno-hit"
                      x={a.rx + off.x}
                      y={a.ry + off.y}
                      width={a.rw}
                      height={a.rh}
                      rx={RECT_CORNER}
                      fill="none"
                      stroke={a.color}
                      stroke-width={isSelected
                        ? RECT_STROKE_W + 1
                        : RECT_STROKE_W}
                      style="cursor:grab; filter:url(#pin-shadow)"
                      on:pointerdown={(e) =>
                        onAnnotationPointerDown(e, a, "body")}
                    />
                    {#if isSelected}<rect
                        x={a.rx + off.x - 2}
                        y={a.ry + off.y - 2}
                        width={a.rw + 4}
                        height={a.rh + 4}
                        rx={RECT_CORNER + 2}
                        fill="none"
                        stroke="white"
                        stroke-width="1"
                        stroke-dasharray="4 3"
                        opacity="0.6"
                      />{/if}
                  {:else}
                    {@const bulb = getBulbCenter(a)}
                    {@const bx = bulb.x + off.x}
                    {@const by = bulb.y + off.y}
                    {@const deg = (a.rotation * 180) / Math.PI}
                    <g
                      transform="translate({bx},{by}) rotate({deg})"
                      filter="url(#pin-shadow)"
                      class="anno-hit"
                    >
                      <path
                        d={teardropPath}
                        fill={a.color}
                        stroke={isSelected ? "white" : "rgba(0,0,0,0.12)"}
                        stroke-width={isSelected ? 2.5 : 1}
                        stroke-linejoin="round"
                      />
                      {#if a.type === "counter"}<g transform="rotate({-deg})"
                          ><text
                            x="0"
                            y="-1"
                            text-anchor="middle"
                            dominant-baseline="middle"
                            fill="white"
                            font-size="15"
                            font-weight="bold"
                            style="user-select:none;pointer-events:none"
                            >{a.count}</text
                          ></g
                        >{/if}
                    </g>
                    <!-- svelte-ignore a11y-no-static-element-interactions -->
                    <circle
                      class="anno-hit"
                      cx={bx}
                      cy={by}
                      r={PIN_R + 5}
                      fill="transparent"
                      style="cursor:grab"
                      on:pointerdown={(e) =>
                        onAnnotationPointerDown(e, a, "bulb")}
                    />
                    {#if isSelected}
                      {@const tx = a.tipX + off.x}
                      {@const ty = a.tipY + off.y}
                      <!-- svelte-ignore a11y-no-static-element-interactions -->
                      <circle
                        class="anno-hit"
                        cx={tx}
                        cy={ty}
                        r="6"
                        fill="white"
                        fill-opacity="0.85"
                        stroke={a.color}
                        stroke-width="2"
                        style="cursor:grab"
                        on:pointerdown={(e) =>
                          onAnnotationPointerDown(e, a, "tip")}
                      />
                    {/if}
                  {/if}
                {/each}
                {#if rectPreview}
                  {@const off = {
                    x: -borderSize,
                    y: -(borderSize + (windowEnabled ? CHROME_H : 0)),
                  }}
                  <rect
                    x={rectPreview.x + off.x}
                    y={rectPreview.y + off.y}
                    width={rectPreview.w}
                    height={rectPreview.h}
                    rx={RECT_CORNER}
                    fill="none"
                    stroke={markerColor}
                    stroke-width={RECT_STROKE_W}
                    stroke-dasharray="6 4"
                    opacity="0.7"
                  />
                {/if}
              </svg>
            </div>
          </div>
        </div>
      {/if}
    </div>

    <div class="footer-divider"></div>
    <footer class="footer">
      <p>Built with Svelte & Vite by KJ</p>
    </footer>
  </div>
</div>

<style>
  :global(body) {
    margin: 0;
    font-family:
      system-ui,
      -apple-system,
      sans-serif;
  }

  .app {
    display: flex;
    height: 100vh;
    background: #09090b;
    color: #d4d4d8;
    overflow: hidden;
  }

  .sidebar {
    flex-shrink: 0;
    width: 14rem;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 1.1rem;
    padding: 1.25rem;
    overflow-y: auto;
    height: 100vh;
  }

  .field {
    display: flex;
    flex-direction: column;
    gap: 0.375rem;
    width: 100%;
    transition: opacity 0.2s;
  }
  .field-inline {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }
  .disabled-field {
    opacity: 0.35;
    pointer-events: none;
  }

  .label {
    font-size: 0.75rem;
    font-weight: 500;
    color: #71717a;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .section-label {
    width: 100%;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.65rem;
    font-weight: 600;
    color: #52525b;
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }
  .section-label::after {
    content: "";
    flex: 1;
    height: 1px;
    background: #27272a;
  }

  .select-wrap {
    position: relative;
  }
  .select-wrap select {
    width: 100%;
    background: #18181b;
    border: 1px solid #27272a;
    color: #d4d4d8;
    font-size: 0.875rem;
    border-radius: 0.5rem;
    padding: 0.375rem 2rem 0.375rem 0.75rem;
    appearance: none;
    -webkit-appearance: none;
    outline: none;
    cursor: pointer;
  }
  .select-wrap select:disabled {
    cursor: not-allowed;
  }
  .select-wrap select:focus {
    border-color: #52525b;
  }
  .chevron {
    position: absolute;
    right: 0.5rem;
    top: 50%;
    transform: translateY(-50%);
    width: 1rem;
    height: 1rem;
    color: #a1a1aa;
    pointer-events: none;
  }
  .select-wrap-sm select {
    padding: 0.3rem 1.75rem 0.3rem 0.6rem;
    font-size: 0.8rem;
  }

  /* BG TABS */

  .bg-tabs {
    gap: 0.5rem;
  }

  .tab-row {
    display: flex;
    background: #18181b;
    border: 1px solid #27272a;
    border-radius: 0.5rem;
    padding: 3px;
    gap: 2px;
  }

  .tab-btn {
    flex: 1;
    font-size: 0.75rem;
    font-weight: 500;
    padding: 0.3rem 0;
    border-radius: 0.35rem;
    border: none;
    background: transparent;
    color: #52525b;
    cursor: pointer;
    transition: all 0.15s;
  }

  .tab-btn:hover {
    color: #a1a1aa;
  }

  .tab-btn.tab-active {
    background: #27272a;
    color: #f4f4f5;
  }

  /* SOLID PANEL */

  .solid-panel {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
  }

  .solid-swatches {
    display: flex;
    flex-wrap: wrap;
    gap: 0.3rem;
    align-items: center;
  }

  .color-swatch {
    width: 1.5rem;
    height: 1.5rem;
    border-radius: 0.3rem;
    border: 2px solid transparent;
    cursor: pointer;
    transition:
      border-color 0.15s,
      transform 0.1s;
    flex-shrink: 0;
  }
  .color-swatch:hover {
    transform: scale(1.12);
  }
  .color-swatch.active {
    border-color: white;
  }

  /* + color picker button */
  .plus-color-wrap {
    width: 1.5rem;
    height: 1.5rem;
    border-radius: 0.3rem;
    border: 1.5px dashed #52525b;
    cursor: pointer;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    transition:
      border-color 0.15s,
      transform 0.1s;
    flex-shrink: 0;
    background: #18181b;
  }
  .plus-color-wrap:hover {
    border-color: #a1a1aa;
    transform: scale(1.12);
  }

  .plus-color-input {
    position: absolute;
    inset: -6px;
    width: calc(100% + 12px);
    height: calc(100% + 12px);
    opacity: 0;
    cursor: pointer;
    border: none;
    padding: 0;
  }

  .plus-icon {
    width: 0.75rem;
    height: 0.75rem;
    color: #71717a;
    pointer-events: none;
    flex-shrink: 0;
  }

  /* GRADIENT PANEL */

  .gradient-panel {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .gradient-swatches {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.35rem;
  }

  .grad-swatch {
    height: 2.25rem;
    border-radius: 0.4rem;
    border: 2px solid transparent;
    cursor: pointer;
    transition:
      border-color 0.15s,
      transform 0.1s;
  }
  .grad-swatch:hover {
    transform: scale(1.04);
  }
  .grad-swatch.active {
    border-color: white;
    box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.2);
  }

  .custom-grad-section {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
  }

  .custom-grad-toggle {
    font-size: 0.75rem;
    font-weight: 500;
    padding: 0.3rem 0.75rem;
    border-radius: 0.4rem;
    border: 1px dashed #3f3f46;
    background: transparent;
    color: #71717a;
    cursor: pointer;
    transition: all 0.15s;
    text-align: center;
    width: 100%;
  }
  .custom-grad-toggle:hover {
    border-color: #71717a;
    color: #a1a1aa;
  }
  .custom-grad-toggle.active {
    border-style: solid;
    border-color: #52525b;
    color: #d4d4d8;
    background: #1c1c20;
  }

  .custom-grad-pickers {
    display: flex;
    align-items: center;
    gap: 0.4rem;
  }

  .custom-grad-picker-wrap {
    flex: 1;
    height: 2rem;
    border-radius: 0.4rem;
    overflow: hidden;
    position: relative;
    cursor: pointer;
    border: 1px solid #3f3f46;
  }

  .grad-color-input {
    position: absolute;
    inset: -4px;
    width: calc(100% + 8px);
    height: calc(100% + 8px);
    opacity: 0;
    cursor: pointer;
    border: none;
    padding: 0;
  }

  .grad-color-preview {
    width: 100%;
    height: 100%;
    pointer-events: none;
  }

  .custom-grad-arrow {
    color: #52525b;
    font-size: 0.875rem;
    flex-shrink: 0;
  }

  .custom-grad-preview {
    height: 2.5rem;
    border-radius: 0.4rem;
    border: 1px solid #27272a;
  }

  /* TOGGLE */
  .toggle {
    position: relative;
    width: 2.5rem;
    height: 1.25rem;
    border-radius: 9999px;
    background: #27272a;
    border: 1px solid #3f3f46;
    cursor: pointer;
    transition: background 0.2s;
    align-self: flex-start;
    padding: 0;
    flex-shrink: 0;
  }
  .toggle.on {
    background: #059669;
  }
  .toggle-dot {
    position: absolute;
    top: 2px;
    left: 2px;
    width: 1rem;
    height: 1rem;
    border-radius: 9999px;
    background: #71717a;
    transition: all 0.2s;
  }
  .toggle-dot.on {
    left: 1.25rem;
    background: white;
  }

  .divider {
    height: 1px;
    background: #27272a;
    width: 100%;
  }

  .marker-colors {
    display: flex;
    gap: 0.35rem;
    align-items: center;
    width: 100%;
  }

  .anno-buttons {
    display: flex;
    gap: 0.35rem;
    width: 100%;
  }

  .tool-btn {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    height: 2.25rem;
    border-radius: 0.5rem;
    border: 1px solid #27272a;
    background: #18181b;
    color: #52525b;
    cursor: not-allowed;
    transition: all 0.15s;
  }
  .tool-btn.enabled {
    background: #27272a;
    color: #d4d4d8;
    border-color: #3f3f46;
    cursor: pointer;
  }
  .tool-btn.enabled:hover {
    background: #3f3f46;
  }
  .tool-btn.tool-active {
    background: #3f3f46 !important;
    border-color: #71717a !important;
    color: white !important;
    cursor: pointer !important;
  }
  .tool-btn.tool-active:hover {
    background: #52525b !important;
  }

  .crop-actions {
    display: flex;
    gap: 0.35rem;
    width: 100%;
  }
  .crop-apply {
    flex: 1;
    background: #059669 !important;
    border-color: #10b981 !important;
    color: white !important;
    cursor: pointer !important;
  }
  .crop-apply:hover {
    background: #047857 !important;
  }
  .crop-cancel {
    flex: 1;
  }

  /* COMPRESS */
  .compress-panel {
    width: 100%;
    background: #111113;
    border: 1px solid #27272a;
    border-radius: 0.5rem;
    padding: 0.65rem 0.75rem;
    display: flex;
    flex-direction: column;
    gap: 0.45rem;
  }
  .compress-quality-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .compress-label {
    font-size: 0.7rem;
    font-weight: 500;
    color: #71717a;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }
  .compress-value {
    font-size: 0.8rem;
    font-weight: 600;
    color: #a1a1aa;
    font-variant-numeric: tabular-nums;
  }
  .quality-slider {
    width: 100%;
    accent-color: #52525b;
    cursor: pointer;
  }
  .compress-hint {
    font-size: 0.68rem;
    color: #52525b;
    text-align: center;
  }
  .compress-apply-btn {
    width: 100%;
    font-size: 0.8rem !important;
    padding: 0.3rem 0.75rem !important;
  }
  .compress-apply-btn.success {
    background: #059669 !important;
    border-color: #10b981 !important;
    color: white !important;
  }
  .compress-apply-btn.error {
    background: #dc2626 !important;
    border-color: #dc2626 !important;
    color: white !important;
  }

  .actions {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    width: 100%;
  }

  .actions-secondary {
    display: flex;
    gap: 0.35rem;
    width: 100%;
  }
  .icon-btn {
    flex: 1;
    font-size: 0.75rem !important;
    padding: 0.3rem 0.5rem !important;
    gap: 0.25rem !important;
  }
  .icon-btn.icon-btn-active.enabled {
    background: #27272a !important;
    border-color: #52525b !important;
    color: #d4d4d8 !important;
  }

  .btn {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.375rem;
    font-size: 0.875rem;
    font-weight: 500;
    padding: 0.375rem 1rem;
    border-radius: 0.5rem;
    border: 1px solid #27272a;
    background: #18181b;
    color: #52525b;
    cursor: not-allowed;
    transition: all 0.2s;
  }
  .btn.enabled {
    background: #27272a;
    color: #d4d4d8;
    border-color: #3f3f46;
    cursor: pointer;
  }
  .btn.enabled:hover {
    background: #3f3f46;
  }

  .copy-btn.enabled {
    background: #f4f4f5;
    color: #18181b;
    border-color: transparent;
  }
  .copy-btn.enabled:hover {
    background: #fff;
  }
  .copy-btn.success {
    background: #059669 !important;
    color: white !important;
    border-color: #059669 !important;
  }
  .copy-btn.error {
    background: #dc2626 !important;
    color: white !important;
    border-color: #dc2626 !important;
  }

  .delete-btn {
    background: #7f1d1d !important;
    border-color: #991b1b !important;
    color: #fca5a5 !important;
    cursor: pointer !important;
    width: 100%;
  }
  .delete-btn:hover {
    background: #991b1b !important;
  }

  .icon {
    width: 1rem;
    height: 1rem;
    flex-shrink: 0;
  }
  .hidden-input {
    display: none;
  }

  .vdivider {
    width: 1px;
    background: #27272a;
    flex-shrink: 0;
    height: 100vh;
  }

  .main-area {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-width: 0;
    min-height: 0;
  }

  .canvas-area {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 0;
    min-width: 0;
    padding: 1.5rem;
    overflow: hidden;
  }

  .prompt {
    border: 2px dashed #27272a;
    border-radius: 1rem;
    padding: 5rem 4rem;
    text-align: center;
  }
  .prompt-title {
    color: #a1a1aa;
    font-size: 1.125rem;
    font-weight: 500;
    margin-bottom: 0.25rem;
  }
  .prompt-sub {
    color: #52525b;
    font-size: 0.875rem;
    margin-bottom: 1rem;
  }
  .prompt-sub kbd {
    padding: 0.125rem 0.375rem;
    background: #27272a;
    border-radius: 0.25rem;
    color: #a1a1aa;
    font-size: 0.75rem;
    font-family: monospace;
  }
  .prompt-or {
    color: #52525b;
    font-size: 0.875rem;
    margin-bottom: 0.75rem;
  }

  .upload-btn {
    display: inline-flex;
    align-items: center;
    gap: 0.375rem;
    font-size: 0.875rem;
    font-weight: 500;
    padding: 0.375rem 1rem;
    border-radius: 0.5rem;
    background: #27272a;
    color: #d4d4d8;
    border: 1px solid #3f3f46;
    cursor: pointer;
    transition: background 0.2s;
  }
  .upload-btn:hover {
    background: #3f3f46;
  }

  .output {
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 0;
    min-width: 0;
    max-height: 100%;
    max-width: 100%;
    overflow: auto;
  }

  .wrapper {
    display: inline-block;
    flex-shrink: 0;
  }
  .tool-cursor {
    cursor: crosshair !important;
  }

  .window-chrome {
    background: #2a2a2e;
    border-radius: 10px 10px 0 0;
    padding: 12px 16px;
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .wdot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
  }
  .wdot.red {
    background: #ff5f57;
  }
  .wdot.yellow {
    background: #febc2e;
  }
  .wdot.green {
    background: #28c840;
  }

  .preview-img {
    display: block;
    object-fit: contain;
    max-width: calc(100vw - 20rem);
    max-height: calc(100vh - 12rem);
    user-select: none;
    -webkit-user-drag: none;
  }

  .annotation-svg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    overflow: visible;
  }
  .annotation-svg .anno-hit {
    pointer-events: auto;
  }
  .annotation-svg rect {
    pointer-events: auto;
  }

  .crop-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 10;
  }
  .crop-shade {
    position: absolute;
    background: rgba(0, 0, 0, 0.55);
    pointer-events: none;
  }
  .crop-shade-top {
    top: 0;
    left: 0;
    right: 0;
  }
  .crop-shade-bottom {
    left: 0;
    right: 0;
  }
  .crop-shade-left {
    left: 0;
  }
  .crop-area {
    position: absolute;
    border: 2px solid white;
    box-shadow: 0 0 0 1px rgba(0, 0, 0, 0.3);
    cursor: move;
    pointer-events: auto;
  }
  .crop-handle {
    position: absolute;
    width: 10px;
    height: 10px;
    background: white;
    border: 1px solid rgba(0, 0, 0, 0.3);
    border-radius: 2px;
    pointer-events: auto;
  }
  .crop-handle.nw {
    top: -5px;
    left: -5px;
    cursor: nw-resize;
  }
  .crop-handle.ne {
    top: -5px;
    right: -5px;
    cursor: ne-resize;
  }
  .crop-handle.sw {
    bottom: -5px;
    left: -5px;
    cursor: sw-resize;
  }
  .crop-handle.se {
    bottom: -5px;
    right: -5px;
    cursor: se-resize;
  }
  .crop-handle.n {
    top: -5px;
    left: 50%;
    transform: translateX(-50%);
    cursor: n-resize;
  }
  .crop-handle.s {
    bottom: -5px;
    left: 50%;
    transform: translateX(-50%);
    cursor: s-resize;
  }
  .crop-handle.w {
    top: 50%;
    left: -5px;
    transform: translateY(-50%);
    cursor: w-resize;
  }
  .crop-handle.e {
    top: 50%;
    right: -5px;
    transform: translateY(-50%);
    cursor: e-resize;
  }

  .footer-divider {
    height: 1px;
    background: #27272a;
    flex-shrink: 0;
  }
  .footer {
    flex-shrink: 0;
    text-align: right;
    padding: 0.6rem 1.25rem;
  }
  .footer p {
    color: #52525b;
    font-size: 0.75rem;
    margin: 0;
  }
</style>
