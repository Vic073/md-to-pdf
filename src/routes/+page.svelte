<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { open, save } from "@tauri-apps/plugin-dialog";
  import { readTextFile, writeTextFile, writeFile } from "@tauri-apps/plugin-fs";
  import { marked } from "marked";
  import DOMPurify from "dompurify";
  import hljs from "highlight.js";
  import jsPDF from "jspdf";
  import html2canvas from "html2canvas";

  // Configure marked with highlight.js
  marked.setOptions({
    // @ts-ignore
    highlight: function (code: string, lang: string) {
      if (lang && hljs.getLanguage(lang)) {
        try {
          return hljs.highlight(code, { language: lang }).value;
        } catch (_) {}
      }
      return hljs.highlightAuto(code).value;
    },
    langPrefix: "hljs language-",
    breaks: true,
    gfm: true,
  });

  let markdownContent = $state("# Welcome to MD to PDF Converter\n\nStart typing in Markdown here...\n\n## Features\n- **Live preview** as you type\n- Export to PDF\n- Import PDF to Markdown\n- Load/Save .md files");
  let previewHtml = $state("");
  let currentFilePath = $state<string | null>(null);
  let statusMessage = $state("Ready");
  let isProcessing = $state(false);

  // Reference to the preview content element
  let previewContentEl: HTMLDivElement | undefined = $state();

  function renderPreview() {
    const raw = marked.parse(markdownContent) as string;
    previewHtml = DOMPurify.sanitize(raw);
  }

  $effect(() => {
    markdownContent;
    renderPreview();
  });

  async function loadFile() {
    try {
      const selected = await open({
        multiple: false,
        filters: [{ name: "Markdown", extensions: ["md"] }],
      });
      if (selected) {
        const content = await readTextFile(selected);
        markdownContent = content;
        currentFilePath = selected;
        statusMessage = `Loaded: ${selected}`;
      }
    } catch (e) {
      statusMessage = `Error loading file: ${e}`;
    }
  }

  async function saveFile() {
    try {
      let filePath = currentFilePath;
      if (!filePath) {
        filePath = await save({
          filters: [{ name: "Markdown", extensions: ["md"] }],
          defaultPath: "document.md",
        });
      }
      if (filePath) {
        await writeTextFile(filePath, markdownContent);
        currentFilePath = filePath;
        statusMessage = `Saved: ${filePath}`;
      }
    } catch (e) {
      statusMessage = `Error saving file: ${e}`;
    }
  }

  async function saveAsFile() {
    try {
      const filePath = await save({
        filters: [{ name: "Markdown", extensions: ["md"] }],
        defaultPath: "document.md",
      });
      if (filePath) {
        await writeTextFile(filePath, markdownContent);
        currentFilePath = filePath;
        statusMessage = `Saved as: ${filePath}`;
      }
    } catch (e) {
      statusMessage = `Error saving file: ${e}`;
    }
  }

  async function savePdf() {
    if (!previewContentEl) {
      statusMessage = "Preview not ready.";
      return;
    }

    // Open save dialog first so user picks destination before we do heavy work
    const suggestedName = `${getFileStem()}-${new Date().toISOString().slice(0, 10)}.pdf`;
    const filePath = await save({
      filters: [{ name: "PDF", extensions: ["pdf"] }],
      defaultPath: suggestedName,
    });
    if (!filePath) return;

    try {
      isProcessing = true;
      statusMessage = "Generating PDF...";

      // Build a light-themed clone of the preview for clean PDF output
      const clone = previewContentEl.cloneNode(true) as HTMLElement;
      clone.style.cssText = `
        position: fixed;
        top: 0; left: 0;
        width: 794px;
        padding: 48px 56px;
        background: #ffffff;
        color: #1a1a2e;
        font-family: 'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif;
        font-size: 13px;
        line-height: 1.7;
        z-index: -9999;
        visibility: hidden;
      `;

      // Override dark styles inside clone
      const style = document.createElement("style");
      style.textContent = `
        h1 { font-size: 1.8em; border-bottom: 2px solid #e94560; padding-bottom: 8px; margin: 24px 0 16px; color: #e94560; }
        h2 { font-size: 1.4em; border-bottom: 1px solid #ddd; padding-bottom: 6px; margin: 20px 0 12px; color: #0f3460; }
        h3 { font-size: 1.2em; margin: 16px 0 8px; color: #533483; }
        h4 { font-size: 1em; margin: 12px 0 8px; color: #333; }
        p { margin: 8px 0; }
        a { color: #0f3460; text-decoration: underline; }
        ul, ol { margin: 8px 0; padding-left: 24px; }
        li { margin: 3px 0; }
        blockquote { border-left: 4px solid #e94560; margin: 12px 0; padding: 8px 16px; background: #f9f9f9; color: #555; }
        code { background: #f0f0f0; padding: 2px 5px; border-radius: 3px; font-size: 0.88em; color: #c7254e; }
        pre { background: #1e1e2e; padding: 14px; border-radius: 6px; overflow: hidden; margin: 12px 0; }
        pre code { background: transparent; color: #cdd6f4; padding: 0; }
        table { border-collapse: collapse; width: 100%; margin: 12px 0; font-size: 12px; }
        th, td { border: 1px solid #ccc; padding: 7px 10px; text-align: left; }
        th { background: #f5f5f5; font-weight: 600; }
        tr:nth-child(even) { background: #fafafa; }
        img { max-width: 100%; }
        hr { border: none; border-top: 1px solid #ddd; margin: 20px 0; }
        strong { color: #c7254e; }
      `;
      clone.insertBefore(style, clone.firstChild);
      document.body.appendChild(clone);

      // Short tick so DOM is painted before canvas capture
      await new Promise((r) => setTimeout(r, 50));

      const canvas = await html2canvas(clone, {
        scale: 2,
        useCORS: true,
        backgroundColor: "#ffffff",
        width: 794,
        windowWidth: 794,
      });

      document.body.removeChild(clone);

      // Build PDF (A4: 210 × 297 mm)
      const pdf = new jsPDF({ orientation: "portrait", unit: "mm", format: "a4" });
      const pageW = pdf.internal.pageSize.getWidth();   // 210
      const pageH = pdf.internal.pageSize.getHeight();  // 297
      const imgW = pageW;
      const imgH = (canvas.height * imgW) / canvas.width;

      const imgData = canvas.toDataURL("image/png");
      let heightLeft = imgH;
      let yPos = 0;

      pdf.addImage(imgData, "PNG", 0, yPos, imgW, imgH);
      heightLeft -= pageH;

      while (heightLeft > 0) {
        yPos -= pageH;
        pdf.addPage();
        pdf.addImage(imgData, "PNG", 0, yPos, imgW, imgH);
        heightLeft -= pageH;
      }

      // Write directly to disk via Tauri
      const pdfBytes = new Uint8Array(pdf.output("arraybuffer") as ArrayBuffer);
      await writeFile(filePath, pdfBytes);

      statusMessage = `PDF saved: ${filePath}`;
    } catch (e) {
      statusMessage = `Error saving PDF: ${e}`;
    } finally {
      isProcessing = false;
    }
  }

  async function importPdf() {
    try {
      const selected = await open({
        multiple: false,
        filters: [{ name: "PDF", extensions: ["pdf"] }],
      });
      if (selected) {
        isProcessing = true;
        statusMessage = "Extracting text from PDF...";
        const extractedText = await invoke<string>("extract_pdf_text", { path: selected });
        const md = textToBasicMarkdown(extractedText);
        markdownContent = md;
        currentFilePath = null;
        statusMessage = `Imported: ${selected}`;
        isProcessing = false;
      }
    } catch (e) {
      statusMessage = `Error importing PDF: ${e}`;
      isProcessing = false;
    }
  }

  function textToBasicMarkdown(text: string): string {
    const lines = text.split("\n");
    const result: string[] = [];
    let consecutiveEmpty = 0;

    for (const line of lines) {
      const trimmed = line.trim();
      if (trimmed === "") {
        consecutiveEmpty++;
        if (consecutiveEmpty <= 2) result.push("");
        continue;
      }
      consecutiveEmpty = 0;

      if (trimmed.length < 60 && trimmed === trimmed.toUpperCase() && trimmed.length > 3 && !/^\d/.test(trimmed)) {
        result.push(`## ${trimmed}`);
        continue;
      }
      if (/^([-*+]|\u2022)\s+/.test(trimmed) || /^\d+[.)]\s+/.test(trimmed)) {
        result.push(trimmed);
        continue;
      }
      result.push(trimmed);
    }
    return result.join("\n");
  }

  function getFileStem(): string {
    const fileName = currentFilePath ? currentFilePath.split(/[/\\]/).pop() : "document";
    return (fileName ?? "document").replace(/\.[^.]+$/, "").replace(/[<>:"/\\|?*\x00-\x1f]/g, "-") || "document";
  }

  function handleKeydown(event: KeyboardEvent) {
    if ((event.ctrlKey || event.metaKey) && event.key === "s") {
      event.preventDefault();
      if (event.shiftKey) {
        saveAsFile();
      } else {
        saveFile();
      }
    }
    if ((event.ctrlKey || event.metaKey) && event.key === "o") {
      event.preventDefault();
      loadFile();
    }
    if ((event.ctrlKey || event.metaKey) && event.key === "p") {
      event.preventDefault();
      savePdf();
    }
  }
</script>

<svelte:window onkeydown={handleKeydown} />

<div class="app">
  <!-- Header / Toolbar -->
  <header class="toolbar">
    <div class="toolbar-left">
      <span class="app-title">MD to PDF Converter</span>
      <span class="file-indicator">{currentFilePath ? currentFilePath.split(/[/\\]/).pop() : "Untitled"}</span>
    </div>
    <div class="toolbar-center">
      <button class="toolbar-btn" onclick={loadFile} title="Open Markdown file (Ctrl+O)">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M22 19a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h5l2 3h9a2 2 0 0 1 2 2z"/></svg>
        Load
      </button>
      <button class="toolbar-btn" onclick={saveFile} title="Save Markdown file (Ctrl+S)">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/><polyline points="17 21 17 13 7 13 7 21"/><polyline points="7 3 7 8 15 8"/></svg>
        Save
      </button>
      <div class="divider"></div>
      <button class="toolbar-btn btn-pdf" onclick={savePdf} title="Save as PDF (Ctrl+P)" disabled={isProcessing}>
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/><polyline points="10 9 9 9 8 9"/></svg>
        {isProcessing ? "Generating…" : "Save PDF"}
      </button>
      <div class="divider"></div>
      <button class="toolbar-btn" onclick={importPdf} title="Import PDF and convert to Markdown" disabled={isProcessing}>
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><path d="M12 18v-6"/><path d="M9 15l3-3 3 3"/></svg>
        PDF to MD
      </button>
    </div>
    <div class="toolbar-right">
      <span class="status">{statusMessage}</span>
    </div>
  </header>

  <!-- Main Content Area -->
  <div class="editor-container">
    <!-- Editor Panel -->
    <div class="editor-panel">
      <div class="panel-header">
        <span class="panel-title">Editor</span>
        <span class="char-count">{markdownContent.length} chars</span>
      </div>
      <textarea
        class="editor-textarea"
        bind:value={markdownContent}
        placeholder="Type your Markdown here..."
        spellcheck="false"
      ></textarea>
    </div>

    <!-- Preview Panel -->
    <div class="preview-panel">
      <div class="panel-header">
        <span class="panel-title">Preview</span>
        <span class="hint">Ctrl+P → Save PDF</span>
      </div>
      <div class="preview-content" bind:this={previewContentEl}>
        {#if previewHtml}
          <div class="markdown-body">{@html previewHtml}</div>
        {:else}
          <div class="empty-preview">
            <p>Nothing to preview</p>
          </div>
        {/if}
      </div>
    </div>
  </div>
</div>

<style>
  :global(*) {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  :global(body) {
    font-family: 'Segoe UI', -apple-system, BlinkMacSystemFont, sans-serif;
    background-color: #0d1117;
    color: #c9d1d9;
    overflow: hidden;
    height: 100vh;
  }

  .app {
    display: flex;
    flex-direction: column;
    height: 100vh;
    overflow: hidden;
  }

  /* ===== TOOLBAR ===== */
  .toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    background: #161b22;
    border-bottom: 1px solid #30363d;
    padding: 6px 16px;
    height: 48px;
    gap: 12px;
    flex-shrink: 0;
  }

  .toolbar-left {
    display: flex;
    align-items: center;
    gap: 10px;
    min-width: 180px;
  }

  .app-title {
    font-weight: 700;
    font-size: 14px;
    color: #e94560;
    white-space: nowrap;
  }

  .file-indicator {
    font-size: 12px;
    color: #8b949e;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 150px;
  }

  .toolbar-center {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .toolbar-btn {
    display: flex;
    align-items: center;
    gap: 6px;
    background: #21262d;
    border: 1px solid #30363d;
    color: #c9d1d9;
    padding: 5px 12px;
    border-radius: 6px;
    font-size: 12px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.15s;
    white-space: nowrap;
  }

  .toolbar-btn:hover {
    background: #30363d;
    border-color: #8b949e;
  }

  .toolbar-btn:active {
    background: #1c2128;
  }

  .toolbar-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .toolbar-btn svg {
    flex-shrink: 0;
  }

  /* PDF button gets a subtle accent */
  .btn-pdf {
    border-color: #e9456040;
    color: #e94560;
  }
  .btn-pdf:hover {
    background: #e9456018;
    border-color: #e94560;
  }

  .divider {
    width: 1px;
    height: 24px;
    background: #30363d;
    margin: 0 6px;
  }

  .toolbar-right {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    min-width: 180px;
  }

  .status {
    font-size: 11px;
    color: #8b949e;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 220px;
  }

  /* ===== EDITOR CONTAINER ===== */
  .editor-container {
    display: flex;
    flex: 1;
    overflow: hidden;
  }

  .editor-panel,
  .preview-panel {
    display: flex;
    flex-direction: column;
    flex: 1;
    min-width: 0;
  }

  .panel-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 6px 16px;
    background: #0d1117;
    border-bottom: 1px solid #21262d;
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: #8b949e;
    flex-shrink: 0;
  }

  .char-count, .hint {
    font-weight: 400;
    font-size: 11px;
    text-transform: none;
    letter-spacing: 0;
  }

  /* ===== EDITOR TEXTAREA ===== */
  .editor-textarea {
    flex: 1;
    width: 100%;
    padding: 20px;
    background: #0d1117;
    color: #c9d1d9;
    border: none;
    outline: none;
    resize: none;
    font-family: 'Cascadia Code', 'Fira Code', 'Consolas', 'Courier New', monospace;
    font-size: 14px;
    line-height: 1.7;
    tab-size: 4;
  }

  .editor-textarea::placeholder {
    color: #484f58;
  }

  /* ===== PREVIEW ===== */
  .preview-panel {
    border-left: 1px solid #21262d;
  }

  .preview-content {
    flex: 1;
    overflow-y: auto;
    padding: 20px;
  }

  .markdown-body {
    font-size: 14px;
    line-height: 1.7;
    max-width: 100%;
    word-wrap: break-word;
  }

  :global(.markdown-body h1) {
    font-size: 1.8em;
    border-bottom: 2px solid #21262d;
    padding-bottom: 8px;
    margin: 24px 0 16px;
    color: #e94560;
    font-weight: 600;
  }

  :global(.markdown-body h2) {
    font-size: 1.4em;
    border-bottom: 1px solid #21262d;
    padding-bottom: 6px;
    margin: 20px 0 12px;
    color: #58a6ff;
    font-weight: 600;
  }

  :global(.markdown-body h3) {
    font-size: 1.2em;
    margin: 16px 0 8px;
    color: #bc8cff;
    font-weight: 600;
  }

  :global(.markdown-body h4) {
    font-size: 1em;
    margin: 12px 0 8px;
    color: #c9d1d9;
    font-weight: 600;
  }

  :global(.markdown-body p) {
    margin: 8px 0;
    line-height: 1.7;
  }

  :global(.markdown-body a) {
    color: #58a6ff;
    text-decoration: none;
  }

  :global(.markdown-body a:hover) {
    text-decoration: underline;
  }

  :global(.markdown-body ul),
  :global(.markdown-body ol) {
    margin: 8px 0;
    padding-left: 24px;
  }

  :global(.markdown-body li) {
    margin: 4px 0;
  }

  :global(.markdown-body blockquote) {
    border-left: 4px solid #e94560;
    margin: 12px 0;
    padding: 8px 16px;
    background: #161b22;
    border-radius: 0 6px 6px 0;
    color: #8b949e;
  }

  :global(.markdown-body code) {
    background: #161b22;
    padding: 2px 6px;
    border-radius: 4px;
    font-family: 'Cascadia Code', 'Fira Code', 'Consolas', monospace;
    font-size: 0.9em;
    color: #f0883e;
  }

  :global(.markdown-body pre) {
    background: #161b22;
    padding: 16px;
    border-radius: 8px;
    overflow-x: auto;
    margin: 12px 0;
    border: 1px solid #21262d;
  }

  :global(.markdown-body pre code) {
    background: transparent;
    padding: 0;
    color: #c9d1d9;
    font-size: 13px;
    line-height: 1.5;
  }

  :global(.markdown-body table) {
    border-collapse: collapse;
    width: 100%;
    margin: 12px 0;
    font-size: 13px;
  }

  :global(.markdown-body th),
  :global(.markdown-body td) {
    border: 1px solid #30363d;
    padding: 8px 12px;
    text-align: left;
  }

  :global(.markdown-body th) {
    background: #161b22;
    font-weight: 600;
    color: #c9d1d9;
  }

  :global(.markdown-body tr:nth-child(even)) {
    background: #0d1117;
  }

  :global(.markdown-body img) {
    max-width: 100%;
    border-radius: 6px;
    margin: 8px 0;
  }

  :global(.markdown-body hr) {
    border: none;
    border-top: 1px solid #30363d;
    margin: 24px 0;
  }

  :global(.markdown-body strong) {
    color: #ffa657;
  }

  :global(.markdown-body em) {
    color: #c9d1d9;
  }

  .empty-preview {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
    color: #484f58;
    font-style: italic;
  }

  /* Scrollbar styles */
  .editor-textarea::-webkit-scrollbar,
  .preview-content::-webkit-scrollbar {
    width: 8px;
  }

  .editor-textarea::-webkit-scrollbar-track,
  .preview-content::-webkit-scrollbar-track {
    background: #0d1117;
  }

  .editor-textarea::-webkit-scrollbar-thumb,
  .preview-content::-webkit-scrollbar-thumb {
    background: #30363d;
    border-radius: 4px;
  }

  .editor-textarea::-webkit-scrollbar-thumb:hover,
  .preview-content::-webkit-scrollbar-thumb:hover {
    background: #484f58;
  }
</style>