<script>
  import { onMount, tick } from "svelte";

  let imgEl;
  let imgSrc = "";
  let imgLoaded = false;
  let fileInput;
  let wrapperEl;

  let originalBlob = null;

  let borderSize = 32;
  let borderRadius = 16;
  let windowEnabled = false;

  let bgMode = "solid";

  let borderColor = "f4f5f6";

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

  let copyState = "idle";

  let compressQuality = 80;
  let showCompressSlider = false;
  let compressState = "idle";
  let originalFileSize = null;
  let compressedFileSize = null;

  let annotations = [];
  let nextCounterId = 1;
  let selectedId = null;
  let activeTool = null;

  let dragging = null;
  let dragOffset = { x: 0, y: 0 };

  let rectStart = null;
  let rectPreview = null;

  let cropStart = null;
  let cropPreview = null;
  let cropRect = null;
  let cropDragging = null;
  let cropDragStart = null;

  let showClearConfirm = false;

  let isDraggingOver = false;

  let sidebarOpen = false;

  let sidebarScrollEl;
  let showScrollHint = false;

  function checkScrollHint() {
    if (!sidebarScrollEl) return;
    const { scrollTop, scrollHeight, clientHeight } = sidebarScrollEl;
    showScrollHint = scrollHeight - scrollTop - clientHeight > 0;
  }

  function onSidebarScroll() {
    checkScrollHint();
  }

  onMount(() => {
    checkScrollHint();
    window.addEventListener("resize", checkScrollHint);
    return () => window.removeEventListener("resize", checkScrollHint);
  });

  $: if (showCompressSlider !== undefined) {
    tick().then(checkScrollHint);
  }

  const BORDER_MIN = 0;
  const BORDER_MAX = 128;
  const RADIUS_MIN = 0;
  const RADIUS_MAX = 128;

  const CHROME_H = 40;
  const CHROME_BG = "#2a2a2e";
  const CHROME_DOTS = ["#ff5f57", "#febc2e", "#28c840"];
  const CHROME_DOT_R = 6;
  const CHROME_DOT_GAP = 20;
  const CHROME_DOT_LEFT = 16;
  const CHROME_CORNER = 10;

  const PALETTE = ["#f97316", "#ef4444", "#3b82f6", "#22c55e", "#a855f7"];
  let markerColor = "#f97316";

  let customMarkerColor = "#ff00ff";

  const PIN_R = 16;
  const PIN_TAIL = 24;
  const RECT_CORNER = 8;
  const RECT_STROKE_W = 3;

  $: hexColor = "#" + (borderColor.replace(/^#/, "").trim() || "f4f5f6");
  $: hasImage = !!imgSrc;

  $: activeGradientColors = useCustomGradient
    ? [customGradientA, customGradientB]
    : (PRESET_GRADIENTS.find((g) => g.id === selectedGradientId)?.colors ??
      PRESET_GRADIENTS[0].colors);

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
    return { x: borderSize, y: borderSize + (windowEnabled ? CHROME_H : 0) };
  }

  function storeOriginal(blob) {
    originalBlob = blob;
  }

  function resetToOriginal() {
    if (!originalBlob) return;
    if (imgSrc.startsWith("blob:")) URL.revokeObjectURL(imgSrc);
    imgLoaded = false;
    imgSrc = URL.createObjectURL(originalBlob);
    annotations = [];
    nextCounterId = 1;
    selectedId = null;
    activeTool = null;
    cropRect = null;
    cropPreview = null;
    showCompressSlider = false;
    compressedFileSize = null;
    compressState = "idle";
    borderSize = 16;
    borderRadius = 16;
    windowEnabled = false;
    bgMode = "solid";
    borderColor = "f4f5f6";
    selectedGradientId = "g1";
    useCustomGradient = false;
  }

  $: hasChanges = hasImage && originalBlob && imgLoaded;

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
    originalFileSize = blob.size;
    compressedFileSize = null;
    storeOriginal(blob);
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

  function handleDragOver(e) {
    e.preventDefault();
    isDraggingOver = true;
  }
  function handleDragLeave() {
    isDraggingOver = false;
  }
  function handleDrop(e) {
    e.preventDefault();
    isDraggingOver = false;
    const files = e.dataTransfer?.files;
    if (files && files.length > 0 && files[0].type.startsWith("image/")) {
      loadBlob(files[0]);
    }
  }

  function requestClear() {
    showClearConfirm = true;
  }
  function confirmClear() {
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
    originalFileSize = null;
    compressedFileSize = null;
    originalBlob = null;
    showClearConfirm = false;
  }
  function cancelClear() {
    showClearConfirm = false;
  }

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
      const cx = e.clientX - rect.left,
        cy = e.clientY - rect.top;
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
        const newBulbX = pos.x - dragOffset.x,
          newBulbY = pos.y - dragOffset.y;
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
      const dx = pos.x - cropDragStart.x,
        dy = pos.y - cropDragStart.y;
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

  function applyCrop() {
    if (!cropRect || !imgEl || !imgLoaded) return;
    const dispW = imgEl.getBoundingClientRect().width,
      dispH = imgEl.getBoundingClientRect().height;
    const sx = imgEl.naturalWidth / dispW,
      sy = imgEl.naturalHeight / dispH;
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

  function cancelCropAction() {
    cropRect = null;
    cropPreview = null;
    cropStart = null;
    activeTool = null;
  }

  function toggleCompressSlider() {
    showCompressSlider = !showCompressSlider;
    compressedFileSize = null;
  }

  function formatBytes(bytes) {
    if (!bytes) return "";
    if (bytes < 1024) return bytes + " B";
    if (bytes < 1048576) return (bytes / 1024).toFixed(1) + " KB";
    return (bytes / 1048576).toFixed(2) + " MB";
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
        compressedFileSize = blob.size;
        if (imgSrc.startsWith("blob:")) URL.revokeObjectURL(imgSrc);
        imgLoaded = false;
        imgSrc = URL.createObjectURL(blob);
        compressState = "success";
        setTimeout(() => {
          compressState = "idle";
          showCompressSlider = false;
        }, 1200);
      },
      "image/jpeg",
      compressQuality / 100,
    );
  }

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

  function handleKeydown(e) {
    if (e.key === "Escape") {
      if (showClearConfirm) {
        showClearConfirm = false;
        return;
      }
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
    if ((e.metaKey || e.ctrlKey) && e.key === "z" && originalBlob) {
      e.preventDefault();
      resetToOriginal();
    }
  }

  function onAppClick(e) {
    if (showClearConfirm && !e.target.closest(".confirm-dialog")) {
      showClearConfirm = false;
    }
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
    if (e.target.closest(".crop-float-actions")) return;
    if (activeTool === "crop") return;
    activeTool = null;
    rectStart = null;
    rectPreview = null;
    selectedId = null;
  }

  function drawBgToCanvas(ctx, w, h) {
    if (bgMode === "gradient") {
      const colors = useCustomGradient
        ? [customGradientA, customGradientB]
        : (PRESET_GRADIENTS.find((g) => g.id === selectedGradientId)?.colors ??
          PRESET_GRADIENTS[0].colors);
      const [c1, c2, c3] =
        colors.length >= 3 ? colors : [colors[0], colors[1], colors[0]];
      ctx.fillStyle = c1;
      ctx.fillRect(0, 0, w, h);
      const g1 = ctx.createRadialGradient(0, 0, 0, 0, 0, w * 0.8);
      g1.addColorStop(0, c1 + "ff");
      g1.addColorStop(1, c1 + "00");
      ctx.fillStyle = g1;
      ctx.fillRect(0, 0, w, h);
      const g2 = ctx.createRadialGradient(w, 0, 0, w, 0, w * 0.8);
      g2.addColorStop(0, c2 + "ff");
      g2.addColorStop(1, c2 + "00");
      ctx.fillStyle = g2;
      ctx.fillRect(0, 0, w, h);
      const c3hex = c3 ?? c1;
      const g3 = ctx.createRadialGradient(w, h, 0, w, h, w * 0.8);
      g3.addColorStop(0, c3hex + "ff");
      g3.addColorStop(1, c3hex + "00");
      ctx.fillStyle = g3;
      ctx.fillRect(0, 0, w, h);
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
    const natW = imgEl.naturalWidth,
      natH = imgEl.naturalHeight;
    const dispW = imgEl.getBoundingClientRect().width,
      dispH = imgEl.getBoundingClientRect().height;
    if (!natW || !natH || !dispW || !dispH) return null;
    const sx = natW / dispW,
      sy = natH / dispH;
    const pad = Math.round(borderSize * sx),
      rad = Math.round(borderRadius * sx);
    const chromeH = windowEnabled ? Math.round(CHROME_H * sx) : 0;
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    canvas.width = natW + pad * 2;
    canvas.height = natH + pad * 2 + chromeH;
    drawBgToCanvas(ctx, canvas.width, canvas.height);
    const imgX = pad,
      imgY = pad + chromeH;
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
      const dr = Math.round(CHROME_DOT_R * sx),
        dg = Math.round(CHROME_DOT_GAP * sx),
        dl = Math.round(CHROME_DOT_LEFT * sx);
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
    const pox = borderSize,
      poy = borderSize + (windowEnabled ? CHROME_H : 0);
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
  <div class="sidebar" class:sidebar-open={sidebarOpen}>
    <div
      class="sidebar-scroll"
      bind:this={sidebarScrollEl}
      on:scroll={onSidebarScroll}
    >
      <div class="section-card">
        <div class="section-label">Border</div>
        <div class="field field-col">
          <div class="slider-header">
            <span class="label">Size</span><span class="slider-value"
              >{borderSize === 0 ? "None" : borderSize + "px"}</span
            >
          </div>
          <input
            type="range"
            min={BORDER_MIN}
            max={BORDER_MAX}
            step="1"
            bind:value={borderSize}
            class="styled-slider"
          />
        </div>
        <div class="field field-col" class:disabled-field={windowEnabled}>
          <div class="slider-header">
            <span class="label">Radius</span><span class="slider-value"
              >{borderRadius === 0 ? "None" : borderRadius + "px"}</span
            >
          </div>
          <input
            type="range"
            min={RADIUS_MIN}
            max={RADIUS_MAX}
            step="1"
            bind:value={borderRadius}
            disabled={windowEnabled}
            class="styled-slider"
          />
        </div>
        <div class="field field-inline">
          <div class="window-label-row">
            <span class="label">Window</span>
            <span class="window-dots-hint"
              ><span class="hint-dot" style="background:#ff5f57"></span><span
                class="hint-dot"
                style="background:#febc2e"
              ></span><span class="hint-dot" style="background:#28c840"
              ></span></span
            >
          </div>
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
      </div>

      <div class="section-card">
        <div class="section-label">Background</div>
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
            <div class="solid-panel">
              <div class="solid-swatches">
                <button
                  class="color-swatch"
                  class:active={borderColor.toLowerCase() === "f4f5f6"}
                  style="background:#f4f5f6; border:1px solid #3f3f46"
                  on:click={() => (borderColor = "f4f5f6")}
                ></button>
                <button
                  class="color-swatch"
                  class:active={borderColor.toLowerCase() === "18181b"}
                  style="background:#18181b; border:1px solid #3f3f46"
                  on:click={() => (borderColor = "18181b")}
                ></button>
                <button
                  class="color-swatch"
                  class:active={"#" + borderColor.replace(/^#/, "") ===
                    "#f97316"}
                  style="background:#f97316"
                  on:click={() => (borderColor = "f97316")}
                ></button>
                <button
                  class="color-swatch"
                  class:active={"#" + borderColor.replace(/^#/, "") ===
                    "#3b82f6"}
                  style="background:#3b82f6"
                  on:click={() => (borderColor = "3b82f6")}
                ></button>
                <div class="custom-color-wrap" title="Pick custom color">
                  <input
                    type="color"
                    value={hexColor}
                    class="custom-color-input"
                    on:input={(e) =>
                      (borderColor = e.target.value.replace("#", ""))}
                  />
                  <svg
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    class="picker-icon"
                    ><path d="M12 2.69l5.66 5.66a8 8 0 1 1-11.31 0z" /></svg
                  >
                </div>
              </div>
            </div>
          {:else}
            <div class="gradient-panel">
              <div class="gradient-swatches">
                {#each PRESET_GRADIENTS as g}
                  <button
                    class="grad-swatch"
                    class:active={selectedGradientId === g.id &&
                      !useCustomGradient}
                    style="background: {meshGradientCSS(g.colors)}"
                    title={g.name}
                    on:click={() => {
                      selectedGradientId = g.id;
                      useCustomGradient = false;
                    }}
                  ></button>
                {/each}
              </div>
              <div class="custom-grad-section">
                <button
                  class="custom-grad-toggle"
                  class:active={useCustomGradient}
                  on:click={() => (useCustomGradient = true)}>Custom</button
                >
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
      </div>

      <div class="section-card">
        <div class="section-label">Annotations</div>
        <div class="marker-colors">
          {#each PALETTE as pc}<button
              class="color-swatch"
              class:active={markerColor === pc}
              style="background:{pc}"
              on:click={() => (markerColor = pc)}
            ></button>{/each}
          <div class="custom-color-wrap" title="Pick custom color">
            <input
              type="color"
              bind:value={customMarkerColor}
              class="custom-color-input"
              on:input={() => (markerColor = customMarkerColor)}
            />
            <svg
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              class="picker-icon"
              ><path d="M12 2.69l5.66 5.66a8 8 0 1 1-11.31 0z" /></svg
            >
          </div>
        </div>
        <div class="anno-buttons">
          <button
            class="tool-btn"
            class:enabled={hasImage}
            class:tool-active={activeTool === "arrow"}
            disabled={!hasImage}
            on:click|stopPropagation={() => toggleTool("arrow")}
          >
            <svg
              width="16"
              height="16"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2.5"
              stroke-linecap="round"
              stroke-linejoin="round"
              ><path d="M12 5V19" /><path d="M5 12l7-7 7 7" /></svg
            >
            <span class="tool-label">Arrow</span>
          </button>
          <button
            class="tool-btn"
            class:enabled={hasImage}
            class:tool-active={activeTool === "counter"}
            disabled={!hasImage}
            on:click|stopPropagation={() => toggleTool("counter")}
            title="Step counter"
          >
            <svg
              width="16"
              height="16"
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
            <span class="tool-label">Count</span>
          </button>
          <button
            class="tool-btn"
            class:enabled={hasImage}
            class:tool-active={activeTool === "rect"}
            disabled={!hasImage}
            on:click|stopPropagation={() => toggleTool("rect")}
            title="Draw a rectangle"
          >
            <svg
              width="16"
              height="16"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              ><rect x="3" y="5" width="18" height="14" rx="3" /></svg
            >
            <span class="tool-label">Box</span>
          </button>
          <button
            class="tool-btn"
            class:enabled={hasImage}
            class:tool-active={activeTool === "crop"}
            disabled={!hasImage}
            on:click|stopPropagation={() => toggleTool("crop")}
          >
            <svg
              width="16"
              height="16"
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
            <span class="tool-label">Crop</span>
          </button>
        </div>
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
            <span>Delete annotation</span>
          </button>
        {/if}
      </div>

      <div class="section-card">
        <div class="section-label">Image</div>
        <div class="misc-row">
          <button
            class="btn icon-btn"
            class:enabled={hasChanges}
            disabled={!hasChanges}
            on:click={resetToOriginal}
            title="Reset to original image"
          >
            <svg
              class="icon"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              ><path d="M3 7v6h6" /><path
                d="M21 17a9 9 0 00-9-9 9 9 0 00-6.69 3L3 13"
              /></svg
            >
            <span>Reset</span>
          </button>
          <button
            class="btn icon-btn trash-btn"
            class:enabled={hasImage}
            disabled={!hasImage}
            on:click|stopPropagation={requestClear}
            title="Delete everything and restart"
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
            <span>Clear</span>
          </button>
        </div>
      </div>
    </div>

    {#if showScrollHint}
      <div
        class="scroll-hint"
        on:click={() =>
          sidebarScrollEl.scrollBy({ top: 80, behavior: "smooth" })}
      >
        <svg
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2.5"
          stroke-linecap="round"
          stroke-linejoin="round"><path d="M6 9l6 6 6-6" /></svg
        >
      </div>
    {/if}

    <div class="sidebar-actions">
      <div class="section-label">Export</div>
      <button
        class="btn misc-compress-btn"
        class:enabled={hasImage && imgLoaded}
        class:icon-btn-active={showCompressSlider}
        disabled={!hasImage || !imgLoaded}
        on:click|stopPropagation={toggleCompressSlider}
      >
        <svg
          class="icon"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><line x1="5" y1="12" x2="19" y2="12" /><path
            d="M12 4v5M9 6l3 3 3-3"
          /><path d="M12 20v-5M9 18l3-3 3 3" /></svg
        >
        <span>Compress</span>
      </button>
      {#if showCompressSlider}
        <div class="compress-panel">
          <div class="compress-quality-row">
            <span class="compress-label">Quality</span><span
              class="compress-value">{compressQuality}%</span
            >
          </div>
          <input
            type="range"
            min="10"
            max="100"
            step="5"
            bind:value={compressQuality}
            class="styled-slider"
          />
          <div class="compress-hint">
            {#if compressQuality >= 85}Highest quality · larger file{:else if compressQuality >= 50}Balanced
              quality &amp; size{:else}Smallest file · visible artifacts{/if}
          </div>
          {#if originalFileSize}
            <div class="compress-sizes">
              <span>Original: {formatBytes(originalFileSize)}</span
              >{#if compressedFileSize}<span
                  >→ {formatBytes(compressedFileSize)}</span
                >{/if}
            </div>
          {/if}
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
            {:else}<span>Apply compression</span>{/if}
          </button>
        </div>
      {/if}
      <div class="export-row">
        <button
          class="btn export-btn"
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
          class="btn export-btn"
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
    <button
      class="sidebar-toggle"
      on:click={() => (sidebarOpen = !sidebarOpen)}
      aria-label="Toggle sidebar"
    >
      {#if sidebarOpen}
        <svg
          width="18"
          height="18"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"><path d="M18 6L6 18M6 6l12 12" /></svg
        >
      {:else}
        <svg
          width="18"
          height="18"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"><path d="M3 12h18M3 6h18M3 18h18" /></svg
        >
      {/if}
    </button>
    {#if sidebarOpen}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <!-- svelte-ignore a11y-no-static-element-interactions -->
      <div
        class="sidebar-backdrop"
        on:click={() => (sidebarOpen = false)}
      ></div>
    {/if}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
      class="canvas-area"
      class:drag-over={isDraggingOver}
      on:click={onCanvasBgClick}
      on:dragover={handleDragOver}
      on:dragleave={handleDragLeave}
      on:drop={handleDrop}
    >
      {#if !hasImage}
        <div class="prompt" class:drag-highlight={isDraggingOver}>
          <div class="prompt-title">Beautify your screenshots</div>
          <div class="prompt-sub">
            Press <kbd
              >{navigator.platform?.includes("Mac") ? "⌘" : "Ctrl"}+V</kbd
            > to paste an image from your clipboard or drag &amp; drop
          </div>
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
            {#if windowEnabled}<div class="window-chrome">
                <div class="wdot red"></div>
                <div class="wdot yellow"></div>
                <div class="wdot green"></div>
              </div>{/if}
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
                  : borderRadius +
                    'px'}; max-width:calc(var(--canvas-w, 100vw - 17rem) - {borderSize *
                  2 +
                  48}px); max-height:calc(100vh - {borderSize * 2 +
                  (windowEnabled ? CHROME_H : 0) +
                  96}px)"
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
                <defs
                  ><filter
                    id="pin-shadow"
                    x="-50%"
                    y="-50%"
                    width="200%"
                    height="200%"
                    ><feDropShadow
                      dx="0"
                      dy="1.5"
                      stdDeviation="2"
                      flood-color="rgba(0,0,0,0.35)"
                    /></filter
                  ></defs
                >
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
        {#if activeTool === "crop" && cropRect}
          <div class="crop-float-actions">
            <button
              class="crop-float-btn crop-float-apply"
              on:click|stopPropagation={applyCrop}
            >
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
              Apply crop
            </button>
            <button
              class="crop-float-btn crop-float-cancel"
              on:click|stopPropagation={cancelCropAction}
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
              Cancel
            </button>
            <span class="crop-float-hint">or Enter / Esc</span>
          </div>
        {/if}
      {/if}
    </div>

    <div class="footer-divider"></div>
    <footer class="footer">
      <p>Built with Svelte &amp; Vite by KJ</p>
    </footer>
  </div>

  {#if showClearConfirm}
    <div class="confirm-overlay">
      <div class="confirm-dialog">
        <div class="confirm-icon">
          <svg
            width="28"
            height="28"
            viewBox="0 0 24 24"
            fill="none"
            stroke="#fca5a5"
            stroke-width="1.5"
            ><path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
            /></svg
          >
        </div>
        <div class="confirm-title">Clear image?</div>
        <div class="confirm-text">
          This will remove your image and all annotations.
        </div>
        <div class="confirm-buttons">
          <button class="confirm-cancel-btn" on:click={cancelClear}
            >Cancel</button
          >
          <button class="confirm-delete-btn" on:click={confirmClear}>
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2.5"
              ><path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
              /></svg
            >
            Clear
          </button>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    margin: 0;
    background: #18191a;
    font-family:
      system-ui,
      -apple-system,
      sans-serif;
  }
  .app {
    --fb-bg: #18191a;
    --fb-card: #242526;
    --fb-card-hover: #3a3b3c;
    --fb-border: #3e4042;
    --fb-text: #e4e6eb;
    --fb-muted: #b0b3b8;
    --fb-dim: #8a8d91;
    --fb-blue: #2374e1;
    display: flex;
    height: 100vh;
    background: var(--fb-bg);
    color: var(--fb-text);
    overflow: hidden;
  }

  .sidebar {
    flex-shrink: 0;
    width: 15rem;
    display: flex;
    flex-direction: column;
    height: 100vh;
    border-right: 1px solid var(--fb-border);
    z-index: 50;
    transition: transform 0.25s ease;
  }
  .sidebar-scroll {
    flex: 1;
    overflow-y: auto;
    padding: 0.75rem 0.75rem 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }
  .sidebar-scroll::before,
  .sidebar-scroll::after {
    content: "";
    flex: 1;
  }
  .sidebar-scroll::-webkit-scrollbar {
    width: 4px;
  }
  .sidebar-scroll::-webkit-scrollbar-thumb {
    background: var(--fb-card-hover);
    border-radius: 2px;
  }

  .section-card {
    background: var(--fb-card);
    border: 1px solid var(--fb-border);
    border-radius: 0.625rem;
    padding: 0.75rem;
    display: flex;
    flex-direction: column;
    gap: 0.625rem;
  }

  .export-row {
    display: flex;
    gap: 0.3rem;
  }
  .export-btn {
    flex: 1;
  }
  .export-btn.enabled {
    background: var(--fb-blue) !important;
    color: #ffffff !important;
    border-color: var(--fb-blue) !important;
  }
  .export-btn.enabled:hover {
    background: #1b64c9 !important;
  }
  .export-btn.success {
    background: #31a24c !important;
    color: white !important;
    border-color: #31a24c !important;
  }
  .export-btn.error {
    background: #dc2626 !important;
    color: white !important;
    border-color: #dc2626 !important;
  }
  .misc-row {
    display: flex;
    gap: 0.3rem;
  }
  .misc-compress-btn {
    width: 100%;
  }
  .misc-compress-btn.icon-btn-active.enabled {
    background: var(--fb-card-hover) !important;
    border-color: var(--fb-dim) !important;
    color: var(--fb-text) !important;
  }
  .compress-panel {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
    background: var(--fb-card);
    border: 1px solid var(--fb-card-hover);
    border-radius: 0.4rem;
    padding: 0.5rem 0.6rem;
  }

  .sidebar-actions {
    flex-shrink: 0;
    padding: 0.75rem 0.75rem 3rem;
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
    border-top: 1px solid var(--fb-border);
    background: var(--fb-bg);
  }

  .scroll-hint {
    flex-shrink: 0;
    display: flex;
    justify-content: center;
    padding: 0.35rem 0;
    color: var(--fb-dim);
    cursor: pointer;
  }
  .scroll-hint:hover {
    color: var(--fb-muted);
  }

  .field {
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
    width: 100%;
    transition: opacity 0.2s;
  }
  .field-col {
    flex-direction: column;
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
    font-size: 0.7rem;
    font-weight: 500;
    color: var(--fb-dim);
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }
  .section-label {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 0.6rem;
    font-weight: 600;
    color: var(--fb-dim);
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }

  .slider-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .slider-value {
    font-size: 0.7rem;
    color: var(--fb-muted);
    font-variant-numeric: tabular-nums;
    font-weight: 600;
  }
  .styled-slider {
    width: 100%;
    height: 4px;
    -webkit-appearance: none;
    appearance: none;
    background: var(--fb-card-hover);
    border-radius: 2px;
    outline: none;
    cursor: pointer;
  }
  .styled-slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px;
    height: 14px;
    border-radius: 50%;
    background: var(--fb-text);
    border: 2px solid var(--fb-bg);
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.4);
  }
  .styled-slider::-moz-range-thumb {
    width: 14px;
    height: 14px;
    border-radius: 50%;
    background: var(--fb-text);
    border: 2px solid var(--fb-bg);
    cursor: pointer;
  }
  .styled-slider:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  .window-label-row {
    display: flex;
    align-items: center;
    gap: 0.4rem;
  }
  .window-dots-hint {
    display: flex;
    gap: 3px;
    align-items: center;
  }
  .hint-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    opacity: 0.6;
  }

  .toggle {
    position: relative;
    width: 2.25rem;
    height: 1.15rem;
    border-radius: 9999px;
    background: var(--fb-card-hover);
    border: 1px solid var(--fb-border);
    cursor: pointer;
    transition: background 0.2s;
    padding: 0;
    flex-shrink: 0;
  }
  .toggle.on {
    background: var(--fb-blue);
  }
  .toggle-dot {
    position: absolute;
    top: 2px;
    left: 2px;
    width: 0.85rem;
    height: 0.85rem;
    border-radius: 9999px;
    background: var(--fb-dim);
    transition: all 0.2s;
  }
  .toggle-dot.on {
    left: 1.1rem;
    background: white;
  }

  .bg-tabs {
    gap: 0.4rem;
  }
  .tab-row {
    display: flex;
    background: var(--fb-card);
    border: 1px solid var(--fb-card-hover);
    border-radius: 0.4rem;
    padding: 2px;
    gap: 2px;
  }
  .tab-btn {
    flex: 1;
    font-size: 0.7rem;
    font-weight: 500;
    padding: 0.25rem 0;
    border-radius: 0.3rem;
    border: none;
    background: transparent;
    color: var(--fb-dim);
    cursor: pointer;
    transition: all 0.15s;
  }
  .tab-btn:hover {
    color: var(--fb-muted);
  }
  .tab-btn.tab-active {
    background: var(--fb-card-hover);
    color: var(--fb-blue);
  }

  .solid-panel {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
  }
  .solid-swatches {
    display: flex;
    flex-wrap: wrap;
    gap: 0.3rem;
    align-items: center;
  }
  .color-swatch {
    width: 1.4rem;
    height: 1.4rem;
    border-radius: 0.25rem;
    border: 2px solid transparent;
    cursor: pointer;
    transition:
      border-color 0.15s,
      transform 0.1s;
    flex-shrink: 0;
  }
  .color-swatch.active {
    border-color: white;
  }

  .custom-color-wrap {
    width: 1.4rem;
    height: 1.4rem;
    border-radius: 0.25rem;
    border: 1.5px dashed var(--fb-dim);
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
    background: var(--fb-card);
  }
  .custom-color-wrap:hover {
    border-color: var(--fb-muted);
  }
  .custom-color-input {
    position: absolute;
    inset: -6px;
    width: calc(100% + 12px);
    height: calc(100% + 12px);
    opacity: 0;
    cursor: pointer;
    border: none;
    padding: 0;
  }
  .picker-icon {
    width: 0.7rem;
    height: 0.7rem;
    color: var(--fb-dim);
    pointer-events: none;
    flex-shrink: 0;
  }

  .gradient-panel {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
  }
  .gradient-swatches {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.3rem;
  }
  .grad-swatch {
    height: 2rem;
    border-radius: 0.35rem;
    border: 2px solid transparent;
    cursor: pointer;
    transition:
      border-color 0.15s,
      transform 0.1s;
  }
  .grad-swatch.active {
    border-color: white;
  }
  .custom-grad-section {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
  }
  .custom-grad-toggle {
    font-size: 0.7rem;
    font-weight: 500;
    padding: 0.25rem 0.6rem;
    border-radius: 0.35rem;
    border: 1px dashed var(--fb-border);
    background: transparent;
    color: var(--fb-dim);
    cursor: pointer;
    transition: all 0.15s;
    text-align: center;
    width: 100%;
  }
  .custom-grad-toggle:hover {
    border-color: var(--fb-dim);
    color: var(--fb-muted);
  }
  .custom-grad-toggle.active {
    border-style: solid;
    border-color: var(--fb-dim);
    color: var(--fb-text);
    background: var(--fb-card-hover);
  }
  .custom-grad-pickers {
    display: flex;
    align-items: center;
    gap: 0.35rem;
  }
  .custom-grad-picker-wrap {
    flex: 1;
    height: 1.75rem;
    border-radius: 0.35rem;
    overflow: hidden;
    position: relative;
    cursor: pointer;
    border: 1px solid var(--fb-border);
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
    color: var(--fb-dim);
    font-size: 0.8rem;
    flex-shrink: 0;
  }
  .custom-grad-preview {
    height: 2rem;
    border-radius: 0.35rem;
    border: 1px solid var(--fb-card-hover);
  }

  .marker-colors {
    display: flex;
    gap: 0.3rem;
    align-items: center;
    width: 100%;
  }
  .anno-buttons {
    display: flex;
    gap: 0.25rem;
    width: 100%;
  }
  .tool-btn {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.15rem;
    height: 2.75rem;
    border-radius: 0.4rem;
    border: 1px solid var(--fb-card-hover);
    background: var(--fb-card);
    color: var(--fb-dim);
    cursor: not-allowed;
    transition: all 0.15s;
    padding: 0.25rem 0;
  }
  .tool-btn.enabled {
    background: var(--fb-border);
    color: var(--fb-muted);
    border-color: var(--fb-card-hover);
    cursor: pointer;
  }
  .tool-btn.enabled:hover {
    background: var(--fb-card-hover);
    color: var(--fb-text);
  }
  .tool-btn.tool-active {
    background: rgba(35, 116, 225, 0.18) !important;
    border-color: var(--fb-blue) !important;
    color: var(--fb-blue) !important;
    cursor: pointer !important;
  }
  .tool-label {
    font-size: 0.55rem;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  .delete-btn {
    background: #7f1d1d !important;
    border-color: #991b1b !important;
    color: #fca5a5 !important;
    cursor: pointer !important;
    width: 100%;
    font-size: 0.75rem !important;
  }
  .delete-btn:hover {
    background: #991b1b !important;
  }

  .compress-quality-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .compress-label {
    font-size: 0.65rem;
    font-weight: 500;
    color: var(--fb-dim);
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }
  .compress-value {
    font-size: 0.75rem;
    font-weight: 600;
    color: var(--fb-muted);
    font-variant-numeric: tabular-nums;
  }
  .compress-hint {
    font-size: 0.6rem;
    color: var(--fb-dim);
    text-align: center;
  }
  .compress-sizes {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.3rem;
    font-size: 0.65rem;
    color: var(--fb-dim);
    font-variant-numeric: tabular-nums;
  }
  .compress-apply-btn {
    width: 100%;
    font-size: 0.75rem !important;
    padding: 0.25rem 0.6rem !important;
  }
  .compress-apply-btn.success {
    background: #31a24c !important;
    border-color: #31a24c !important;
    color: white !important;
  }
  .compress-apply-btn.error {
    background: #dc2626 !important;
    border-color: #dc2626 !important;
    color: white !important;
  }

  .btn {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.3rem;
    font-size: 0.8rem;
    font-weight: 500;
    padding: 0.35rem 0.75rem;
    border-radius: 0.4rem;
    border: 1px solid var(--fb-card-hover);
    background: var(--fb-card);
    color: var(--fb-dim);
    cursor: not-allowed;
    transition: all 0.2s;
  }
  .btn.enabled {
    background: var(--fb-card-hover);
    color: var(--fb-text);
    border-color: var(--fb-border);
    cursor: pointer;
  }
  .btn.enabled:hover {
    background: var(--fb-border);
  }
  .icon-btn {
    flex: 1;
    font-size: 0.65rem !important;
    padding: 0.25rem 0.35rem !important;
    gap: 0.2rem !important;
  }
  .icon-btn.icon-btn-active.enabled {
    background: var(--fb-card-hover) !important;
    border-color: var(--fb-dim) !important;
    color: var(--fb-text) !important;
  }
  .trash-btn.enabled {
    background: #2a1215 !important;
    border-color: #7f1d1d !important;
    color: #fca5a5 !important;
  }
  .trash-btn.enabled:hover {
    background: #3b1219 !important;
  }
  .icon {
    width: 0.9rem;
    height: 0.9rem;
    flex-shrink: 0;
  }
  .hidden-input {
    display: none;
  }
  .vdivider {
    width: 1px;
    background: var(--fb-border);
    flex-shrink: 0;
  }

  .main-area {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-width: 0;
    min-height: 0;
    position: relative;
  }
  .canvas-area {
    --canvas-w: 100vw - 17rem;
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 0;
    min-width: 0;
    padding: 1.5rem;
    overflow: hidden;
    position: relative;
    transition: background 0.2s;
  }
  .canvas-area.drag-over {
    background: rgba(59, 130, 246, 0.05);
  }

  .prompt {
    border: 2px dashed var(--fb-card-hover);
    border-radius: 1rem;
    padding: 3rem 4rem;
    text-align: center;
    transition:
      border-color 0.2s,
      background 0.2s;
  }
  .prompt.drag-highlight {
    border-color: var(--fb-blue);
    background: rgba(59, 130, 246, 0.05);
  }
  .prompt-title {
    color: var(--fb-muted);
    font-size: 1.125rem;
    font-weight: 500;
    margin-bottom: 0.25rem;
  }
  .prompt-sub {
    color: var(--fb-dim);
    font-size: 0.8rem;
    margin-bottom: 0.5rem;
  }
  .prompt-sub kbd {
    padding: 0.1rem 0.3rem;
    background: var(--fb-card-hover);
    border-radius: 0.2rem;
    color: var(--fb-muted);
    font-size: 0.7rem;
    font-family: monospace;
  }
  .prompt-or {
    color: var(--fb-dim);
    font-size: 0.8rem;
    margin-bottom: 0.5rem;
  }
  .upload-btn {
    display: inline-flex;
    align-items: center;
    gap: 0.3rem;
    font-size: 0.8rem;
    font-weight: 500;
    padding: 0.35rem 0.85rem;
    border-radius: 0.4rem;
    background: var(--fb-blue);
    color: #ffffff;
    border: 1px solid var(--fb-blue);
    cursor: pointer;
    transition: background 0.2s;
  }
  .upload-btn:hover {
    background: #1b64c9;
  }

  .footer-divider {
    height: 1px;
    background: var(--fb-border);
    flex-shrink: 0;
  }
  .footer {
    flex-shrink: 0;
    text-align: right;
    padding: 0.6rem 1.25rem;
  }
  .footer p {
    color: var(--fb-dim);
    font-size: 0.75rem;
    margin: 0;
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
    background: var(--fb-card-hover);
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

  .crop-float-actions {
    position: absolute;
    bottom: 1.25rem;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    align-items: center;
    gap: 0.4rem;
    background: var(--fb-card);
    border: 1px solid var(--fb-border);
    border-radius: 0.6rem;
    padding: 0.4rem 0.6rem;
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.5);
    z-index: 20;
  }
  .crop-float-btn {
    display: flex;
    align-items: center;
    gap: 0.25rem;
    font-size: 0.75rem;
    font-weight: 500;
    padding: 0.3rem 0.6rem;
    border-radius: 0.35rem;
    border: none;
    cursor: pointer;
    transition: all 0.15s;
  }
  .crop-float-apply {
    background: #31a24c;
    color: white;
  }
  .crop-float-apply:hover {
    background: #2f8d46;
  }
  .crop-float-cancel {
    background: var(--fb-card-hover);
    color: var(--fb-text);
  }
  .crop-float-cancel:hover {
    background: var(--fb-border);
  }
  .crop-float-hint {
    font-size: 0.6rem;
    color: var(--fb-dim);
    margin-left: 0.25rem;
  }

  .confirm-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.6);
    z-index: 200;
    display: flex;
    align-items: center;
    justify-content: center;
    backdrop-filter: blur(4px);
  }
  .confirm-dialog {
    background: var(--fb-card);
    border: 1px solid var(--fb-border);
    border-radius: 0.75rem;
    padding: 1.5rem;
    max-width: 20rem;
    text-align: center;
    box-shadow: 0 16px 48px rgba(0, 0, 0, 0.6);
  }
  .confirm-icon {
    margin-bottom: 0.75rem;
  }
  .confirm-title {
    font-size: 1rem;
    font-weight: 600;
    color: var(--fb-text);
    margin-bottom: 0.35rem;
  }
  .confirm-text {
    font-size: 0.8rem;
    color: var(--fb-dim);
    margin-bottom: 1.25rem;
    line-height: 1.4;
  }
  .confirm-buttons {
    display: flex;
    gap: 0.5rem;
  }
  .confirm-cancel-btn {
    flex: 1;
    padding: 0.4rem 0.75rem;
    border-radius: 0.4rem;
    border: 1px solid var(--fb-border);
    background: var(--fb-card-hover);
    color: var(--fb-text);
    font-size: 0.8rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.15s;
  }
  .confirm-cancel-btn:hover {
    background: var(--fb-border);
  }
  .confirm-delete-btn {
    flex: 1;
    padding: 0.4rem 0.75rem;
    border-radius: 0.4rem;
    border: 1px solid #991b1b;
    background: #7f1d1d;
    color: #fca5a5;
    font-size: 0.8rem;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.15s;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.3rem;
  }
  .confirm-delete-btn:hover {
    background: #991b1b;
  }

  .sidebar-toggle {
    display: none;
  }
  .sidebar-backdrop {
    display: none;
  }

  @media (max-width: 960px) {
    .sidebar {
      width: 13.5rem;
    }
    .canvas-area {
      --canvas-w: 100vw - 15rem;
      padding: 1rem;
    }
    .prompt {
      padding: 2rem 2.5rem;
    }
  }

  @media (max-width: 720px) {
    .sidebar {
      position: fixed;
      top: 0;
      left: 0;
      bottom: 0;
      width: 16rem;
      transform: translateX(-100%);
      background: var(--fb-bg);
      box-shadow: 4px 0 24px rgba(0, 0, 0, 0.5);
    }
    .sidebar.sidebar-open {
      transform: translateX(0);
    }
    .vdivider {
      display: none;
    }
    .sidebar-toggle {
      display: flex;
      align-items: center;
      justify-content: center;
      position: fixed;
      top: 0.6rem;
      left: 0.6rem;
      z-index: 60;
      width: 2.25rem;
      height: 2.25rem;
      border-radius: 0.5rem;
      background: var(--fb-card);
      border: 1px solid var(--fb-card-hover);
      color: var(--fb-muted);
      cursor: pointer;
      transition: background 0.15s;
    }
    .sidebar-toggle:hover {
      background: var(--fb-card-hover);
    }
    .sidebar-backdrop {
      display: block;
      position: fixed;
      inset: 0;
      z-index: 40;
      background: rgba(0, 0, 0, 0.5);
      backdrop-filter: blur(2px);
    }
    .canvas-area {
      --canvas-w: 100vw;
      padding: 0.75rem;
      padding-top: 3rem;
    }
    .main-area {
      height: 100vh;
    }
    .prompt {
      padding: 2rem 1.5rem;
    }
    .prompt-title {
      font-size: 1rem;
    }
    .crop-float-actions {
      bottom: 0.75rem;
      padding: 0.35rem 0.5rem;
    }
    .footer {
      padding: 0.4rem 0.75rem;
    }
    .footer p {
      font-size: 0.65rem;
    }
  }

  @media (max-width: 480px) {
    .sidebar {
      width: 14rem;
    }
    .section-card {
      padding: 0.6rem;
      gap: 0.5rem;
    }
    .sidebar-scroll {
      padding: 0.5rem;
      gap: 0.4rem;
    }
    .sidebar-actions {
      padding: 0.5rem 0.5rem 1rem;
    }
    .tool-btn {
      height: 2.5rem;
    }
    .canvas-area {
      padding: 0.5rem;
      padding-top: 2.75rem;
    }
    .prompt {
      padding: 1.5rem 1rem;
    }
  }
</style>