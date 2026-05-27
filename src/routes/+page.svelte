<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { open, save } from "@tauri-apps/plugin-dialog";
  import { readTextFile, writeTextFile, writeFile } from "@tauri-apps/plugin-fs";
  import { marked } from "marked";
  import DOMPurify from "dompurify";
  import hljs from "highlight.js";
  // @ts-ignore
  import pdfMake from "pdfmake/build/pdfmake";
  // @ts-ignore
  import * as pdfFonts from "pdfmake/build/vfs_fonts";

  const vfs = pdfFonts?.pdfMake?.vfs || (pdfFonts as any)?.default?.pdfMake?.vfs || (pdfFonts as any)?.default?.vfs || (pdfFonts as any)?.vfs || pdfFonts;
  if (vfs) {
    pdfMake.vfs = vfs;
  }
  if (typeof (pdfMake as any).addVirtualFileSystem === "function") {
    try {
      (pdfMake as any).addVirtualFileSystem(pdfFonts);
    } catch (_) {}
  }

  // ─── Marked setup ────────────────────────────────────────────────────────────
  marked.setOptions({
    // @ts-ignore
    highlight: (code: string, lang: string) => {
      if (lang && hljs.getLanguage(lang)) {
        try { return hljs.highlight(code, { language: lang }).value; } catch (_) {}
      }
      return hljs.highlightAuto(code).value;
    },
    langPrefix: "hljs language-",
    breaks: true,
    gfm: true,
  });

  // ─── State ───────────────────────────────────────────────────────────────────
  let markdownContent = $state(
    "# Welcome to MD to PDF Converter\n\nStart typing in Markdown here…\n\n## Features\n- **Live preview** as you type\n- Save as PDF (real text, not screenshots)\n- Import PDF to Markdown\n- Load/Save `.md` files\n\n```js\nconsole.log('hello world');\n```"
  );
  let previewHtml = $state("");
  let currentFilePath = $state<string | null>(null);
  let statusMessage = $state("Ready");
  let isProcessing = $state(false);

  // ─── Preview rendering ───────────────────────────────────────────────────────
  function renderPreview() {
    const raw = marked.parse(markdownContent) as string;
    previewHtml = DOMPurify.sanitize(raw);
  }

  $effect(() => { markdownContent; renderPreview(); });

  // ─── File operations ─────────────────────────────────────────────────────────
  async function loadFile() {
    try {
      const selected = await open({ multiple: false, filters: [{ name: "Markdown", extensions: ["md"] }] });
      if (selected) {
        markdownContent = await readTextFile(selected);
        currentFilePath = selected;
        statusMessage = `Loaded: ${fileName(selected)}`;
      }
    } catch (e) { statusMessage = `Error loading: ${e}`; }
  }

  async function saveFile() {
    try {
      let fp = currentFilePath;
      if (!fp) fp = await save({ filters: [{ name: "Markdown", extensions: ["md"] }], defaultPath: "document.md" });
      if (fp) {
        await writeTextFile(fp, markdownContent);
        currentFilePath = fp;
        statusMessage = `Saved: ${fileName(fp)}`;
      }
    } catch (e) { statusMessage = `Error saving: ${e}`; }
  }

  async function saveAsFile() { currentFilePath = null; await saveFile(); }

  // ─── Markdown → pdfmake converter ───────────────────────────────────────────
  type PdfItem = any;

  function inlineTokens(tokens: any[]): PdfItem[] {
    const out: PdfItem[] = [];
    for (const t of tokens) {
      switch (t.type) {
        case "text":
          if (t.tokens?.length) out.push(...inlineTokens(t.tokens));
          else out.push(t.text ?? "");
          break;
        case "strong":
          out.push({ text: inlineTokens(t.tokens ?? [{ type: "text", text: t.text }]), bold: true });
          break;
        case "em":
          out.push({ text: inlineTokens(t.tokens ?? [{ type: "text", text: t.text }]), italics: true });
          break;
        case "codespan":
          out.push({ text: t.text, font: "Courier", fontSize: 9.5, color: "#c7254e" });
          break;
        case "link":
          out.push({ text: t.text || t.href, color: "#0066cc", decoration: "underline" });
          break;
        case "br":
          out.push("\n");
          break;
        case "del":
          out.push({ text: inlineTokens(t.tokens ?? []), decoration: "lineThrough" });
          break;
        default:
          if (t.raw) out.push(t.raw);
          break;
      }
    }
    return out;
  }

  function blockTokens(tokens: any[]): PdfItem[] {
    const out: PdfItem[] = [];
    for (const t of tokens) {
      switch (t.type) {

        case "heading": {
          const sizes:  Record<number,number> = {1:22,2:18,3:15,4:13,5:12,6:11};
          const colors: Record<number,string> = {1:"#c0392b",2:"#1a5276",3:"#6c3483",4:"#222",5:"#333",6:"#555"};
          const top:    Record<number,number> = {1:18,2:14,3:11,4:9,5:7,6:6};
          out.push({
            text: inlineTokens(t.tokens ?? [{ type: "text", text: t.text }]),
            fontSize: sizes[t.depth] ?? 11,
            bold: true,
            color: colors[t.depth] ?? "#222",
            margin: [0, top[t.depth] ?? 8, 0, 4],
          });
          if (t.depth <= 2) {
            out.push({
              canvas: [{
                type: "line", x1: 0, y1: 0, x2: 497, y2: 0,
                lineWidth: t.depth === 1 ? 1.5 : 0.75,
                lineColor: t.depth === 1 ? "#e74c3c" : "#cccccc",
              }],
              margin: [0, 0, 0, 6],
            });
          }
          break;
        }

        case "paragraph": {
          out.push({
            text: inlineTokens(t.tokens ?? [{ type: "text", text: t.text }]),
            margin: [0, 3, 0, 3],
          });
          break;
        }

        case "code": {
          // Wrap long lines so they don't overflow the page
          const wrapped = t.text
            .split("\n")
            .map((line: string) => line.length > 88 ? line.match(/.{1,88}/g)!.join("\n") : line)
            .join("\n");
          out.push({
            table: {
              widths: ["*"],
              body: [[{
                text: wrapped,
                font: "Courier",
                fontSize: 8.5,
                color: "#abb2bf",
                fillColor: "#282c34",
                border: [false, false, false, false],
                margin: [8, 7, 8, 7],
                lineHeight: 1.35,
                preserveLeadingSpaces: true,
              }]],
            },
            layout: "noBorders",
            margin: [0, 6, 0, 8],
          });
          break;
        }

        case "blockquote": {
          const inner = blockTokens(t.tokens ?? []);
          out.push({
            table: {
              widths: [3, "*"],
              body: [[
                { text: "", fillColor: "#e74c3c", border: [false,false,false,false] },
                {
                  stack: inner.length ? inner : [{ text: "" }],
                  border: [false,false,false,false],
                  color: "#555555",
                  italics: true,
                  margin: [10, 0, 0, 0],
                },
              ]],
            },
            layout: {
              defaultBorder: false,
              paddingLeft: () => 0, paddingRight: () => 0,
              paddingTop: () => 5, paddingBottom: () => 5,
            },
            margin: [0, 6, 0, 6],
          });
          break;
        }

        case "list": {
          const items = t.items.map((item: any) => {
            const inner = blockTokens(item.tokens ?? []);
            return inner.length === 1 ? inner[0] : { stack: inner };
          });
          out.push(t.ordered
            ? { ol: items, margin: [0, 4, 0, 4] }
            : { ul: items, margin: [0, 4, 0, 4] }
          );
          break;
        }

        case "hr": {
          out.push({
            canvas: [{ type:"line", x1:0, y1:0, x2:497, y2:0, lineWidth:0.5, lineColor:"#cccccc" }],
            margin: [0, 12, 0, 12],
          });
          break;
        }

        case "table": {
          const headerCells = t.header.map((cell: any) => ({
            text: inlineTokens(cell.tokens ?? [{ type:"text", text: cell.text }]),
            bold: true, fillColor: "#ecf0f1", fontSize: 9.5,
          }));
          const bodyRows = t.rows.map((row: any) =>
            row.map((cell: any) => ({
              text: inlineTokens(cell.tokens ?? [{ type:"text", text: cell.text }]),
              fontSize: 9.5,
            }))
          );
          out.push({
            table: { headerRows: 1, body: [headerCells, ...bodyRows] },
            layout: "lightHorizontalLines",
            margin: [0, 8, 0, 8],
          });
          break;
        }

        case "space":
          out.push({ text: " ", margin: [0, 2, 0, 2] });
          break;

        default:
          if (t.text) out.push({ text: t.text, margin: [0, 3, 0, 3] });
          break;
      }
    }
    return out;
  }

  // ─── Save PDF ────────────────────────────────────────────────────────────────
  async function savePdf() {
    const defaultName = `${fileStem()}-${new Date().toISOString().slice(0, 10)}.pdf`;
    const fp = await save({ filters: [{ name: "PDF", extensions: ["pdf"] }], defaultPath: defaultName });
    if (!fp) return;

    try {
      isProcessing = true;
      statusMessage = "Building PDF…";

      const docDef = {
        pageSize: "A4",
        pageMargins: [57, 57, 57, 57],
        content: blockTokens(marked.lexer(markdownContent)),
        defaultStyle: { font: "Roboto", fontSize: 11, lineHeight: 1.45, color: "#1a1a2e" },
      };

      const pdfBytes = await new Promise<Uint8Array>((resolve, reject) => {
        const timeout = setTimeout(() => {
          reject(new Error("PDF generation timed out after 10 seconds."));
        }, 10000);

        try {
          const pdfDoc = pdfMake.createPdf(docDef, undefined, undefined, vfs);
          pdfDoc.getBuffer((buf: any) => {
            clearTimeout(timeout);
            if (buf) {
              resolve(buf instanceof Uint8Array ? buf : new Uint8Array(buf));
            } else {
              reject(new Error("Failed to retrieve generated PDF buffer."));
            }
          });
        } catch (err) {
          clearTimeout(timeout);
          reject(err);
        }
      });

      await writeFile(fp, pdfBytes);
      statusMessage = `PDF saved — ${(pdfBytes.length / 1024).toFixed(0)} KB`;
    } catch (e) {
      statusMessage = `PDF error: ${e instanceof Error ? e.message : e}`;
      console.error(e);
    } finally {
      isProcessing = false;
    }
  }

  // ─── PDF → Markdown ──────────────────────────────────────────────────────────
  async function importPdf() {
    try {
      const selected = await open({ multiple: false, filters: [{ name: "PDF", extensions: ["pdf"] }] });
      if (!selected) return;
      isProcessing = true;
      statusMessage = "Extracting text…";
      const text = await invoke<string>("extract_pdf_text", { path: selected });
      markdownContent = textToMarkdown(text);
      currentFilePath = null;
      statusMessage = `Imported: ${fileName(selected)}`;
    } catch (e) {
      statusMessage = `Import error: ${e}`;
    } finally { isProcessing = false; }
  }

  function textToMarkdown(text: string): string {
    const lines = text.split("\n");
    const result: string[] = [];
    let emptyCount = 0;
    for (const line of lines) {
      const t = line.trim();
      if (!t) { if (++emptyCount <= 2) result.push(""); continue; }
      emptyCount = 0;
      if (t.length < 60 && t === t.toUpperCase() && t.length > 3 && !/^\d/.test(t)) { result.push(`## ${t}`); continue; }
      if (/^([-*+]|\u2022)\s+/.test(t) || /^\d+[.)]\s+/.test(t)) { result.push(t); continue; }
      result.push(t);
    }
    return result.join("\n");
  }

  // ─── Helpers ─────────────────────────────────────────────────────────────────
  function fileName(p: string) { return p.split(/[/\\]/).pop() ?? p; }
  function fileStem() {
    const n = currentFilePath ? fileName(currentFilePath) : "document";
    return n.replace(/\.[^.]+$/, "").replace(/[<>:"/\\|?*\x00-\x1f]/g, "-") || "document";
  }
  function handleKeydown(e: KeyboardEvent) {
    if (!(e.ctrlKey || e.metaKey)) return;
    if (e.key === "s") { e.preventDefault(); e.shiftKey ? saveAsFile() : saveFile(); }
    if (e.key === "o") { e.preventDefault(); loadFile(); }
    if (e.key === "p") { e.preventDefault(); savePdf(); }
  }
</script>

<svelte:window onkeydown={handleKeydown} />

<div class="app">
  <header class="toolbar">
    <div class="toolbar-left">
      <span class="app-title">MD ↔ PDF</span>
      <span class="file-indicator">{currentFilePath ? fileName(currentFilePath) : "Untitled"}</span>
    </div>
    <div class="toolbar-center">
      <button class="toolbar-btn" onclick={loadFile} title="Open .md (Ctrl+O)">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M22 19a2 2 0 0 1-2 2H4a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h5l2 3h9a2 2 0 0 1 2 2z"/></svg>
        Load
      </button>
      <button class="toolbar-btn" onclick={saveFile} title="Save .md (Ctrl+S)">
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/><polyline points="17 21 17 13 7 13 7 21"/><polyline points="7 3 7 8 15 8"/></svg>
        Save
      </button>
      <div class="sep"></div>
      <button class="toolbar-btn btn-pdf" onclick={savePdf} title="Save PDF (Ctrl+P)" disabled={isProcessing}>
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/></svg>
        {isProcessing ? "Building…" : "Save PDF"}
      </button>
      <div class="sep"></div>
      <button class="toolbar-btn" onclick={importPdf} title="Import PDF → Markdown" disabled={isProcessing}>
        <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><path d="M12 18v-6"/><path d="M9 15l3-3 3 3"/></svg>
        PDF → MD
      </button>
    </div>
    <div class="toolbar-right">
      <span class="status">{statusMessage}</span>
    </div>
  </header>

  <div class="editor-container">
    <div class="editor-panel">
      <div class="panel-header">
        <span>Editor</span>
        <span class="dim">{markdownContent.length.toLocaleString()} chars</span>
      </div>
      <textarea class="editor-textarea" bind:value={markdownContent} placeholder="Type Markdown here…" spellcheck="false"></textarea>
    </div>
    <div class="preview-panel">
      <div class="panel-header">
        <span>Preview</span>
        <span class="dim">Ctrl+P → Save PDF</span>
      </div>
      <div class="preview-content">
        {#if previewHtml}
          <div class="markdown-body">{@html previewHtml}</div>
        {:else}
          <div class="empty-preview"><p>Nothing to preview yet</p></div>
        {/if}
      </div>
    </div>
  </div>
</div>

<style>
  :global(*) { margin:0; padding:0; box-sizing:border-box; }
  :global(body) { font-family:'Segoe UI',-apple-system,BlinkMacSystemFont,sans-serif; background:#0d1117; color:#c9d1d9; overflow:hidden; height:100vh; }
  .app { display:flex; flex-direction:column; height:100vh; overflow:hidden; }

  .toolbar { display:flex; align-items:center; justify-content:space-between; background:#161b22; border-bottom:1px solid #30363d; padding:6px 16px; height:46px; gap:12px; flex-shrink:0; }
  .toolbar-left { display:flex; align-items:center; gap:10px; min-width:160px; }
  .app-title { font-weight:700; font-size:14px; color:#e94560; white-space:nowrap; }
  .file-indicator { font-size:11px; color:#8b949e; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; max-width:160px; }
  .toolbar-center { display:flex; align-items:center; gap:4px; }
  .toolbar-right { display:flex; align-items:center; justify-content:flex-end; min-width:160px; }
  .status { font-size:11px; color:#8b949e; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; max-width:220px; }

  .toolbar-btn { display:flex; align-items:center; gap:5px; background:#21262d; border:1px solid #30363d; color:#c9d1d9; padding:5px 11px; border-radius:6px; font-size:12px; font-weight:500; cursor:pointer; transition:background .12s,border-color .12s; white-space:nowrap; }
  .toolbar-btn:hover { background:#30363d; border-color:#8b949e; }
  .toolbar-btn:active { background:#1c2128; }
  .toolbar-btn:disabled { opacity:.45; cursor:not-allowed; }
  .btn-pdf { color:#e94560; border-color:rgba(233,69,96,.35); }
  .btn-pdf:hover { background:rgba(233,69,96,.12); border-color:#e94560; }
  .sep { width:1px; height:22px; background:#30363d; margin:0 4px; }

  .editor-container { display:flex; flex:1; overflow:hidden; }
  .editor-panel, .preview-panel { display:flex; flex-direction:column; flex:1; min-width:0; }
  .preview-panel { border-left:1px solid #21262d; }

  .panel-header { display:flex; align-items:center; justify-content:space-between; padding:5px 16px; background:#0d1117; border-bottom:1px solid #21262d; font-size:10px; font-weight:700; text-transform:uppercase; letter-spacing:.6px; color:#8b949e; flex-shrink:0; }
  .dim { font-weight:400; text-transform:none; letter-spacing:0; font-size:10px; }

  .editor-textarea { flex:1; width:100%; padding:20px; background:#0d1117; color:#c9d1d9; border:none; outline:none; resize:none; font-family:'Cascadia Code','Fira Code',Consolas,monospace; font-size:13.5px; line-height:1.7; tab-size:2; }
  .editor-textarea::placeholder { color:#484f58; }

  .preview-content { flex:1; overflow-y:auto; padding:20px 24px; }
  .markdown-body { font-size:14px; line-height:1.7; word-wrap:break-word; }

  :global(.markdown-body h1){font-size:1.8em;border-bottom:2px solid #21262d;padding-bottom:8px;margin:24px 0 12px;color:#e94560;font-weight:700}
  :global(.markdown-body h2){font-size:1.4em;border-bottom:1px solid #21262d;padding-bottom:5px;margin:20px 0 10px;color:#58a6ff;font-weight:600}
  :global(.markdown-body h3){font-size:1.2em;margin:16px 0 8px;color:#bc8cff;font-weight:600}
  :global(.markdown-body h4){font-size:1em;margin:12px 0 6px;color:#c9d1d9;font-weight:600}
  :global(.markdown-body p){margin:6px 0}
  :global(.markdown-body a){color:#58a6ff;text-decoration:none}
  :global(.markdown-body a:hover){text-decoration:underline}
  :global(.markdown-body ul),:global(.markdown-body ol){margin:6px 0;padding-left:22px}
  :global(.markdown-body li){margin:3px 0}
  :global(.markdown-body blockquote){border-left:4px solid #e94560;margin:10px 0;padding:6px 14px;background:#161b22;border-radius:0 6px 6px 0;color:#8b949e}
  :global(.markdown-body code){background:#161b22;padding:2px 5px;border-radius:4px;font-family:'Cascadia Code','Fira Code',monospace;font-size:.88em;color:#f0883e}
  :global(.markdown-body pre){background:#161b22;padding:14px;border-radius:8px;overflow-x:auto;margin:10px 0;border:1px solid #21262d}
  :global(.markdown-body pre code){background:transparent;padding:0;color:#c9d1d9;font-size:12.5px;line-height:1.5}
  :global(.markdown-body table){border-collapse:collapse;width:100%;margin:10px 0;font-size:13px}
  :global(.markdown-body th),:global(.markdown-body td){border:1px solid #30363d;padding:7px 11px;text-align:left}
  :global(.markdown-body th){background:#161b22;font-weight:600}
  :global(.markdown-body tr:nth-child(even)){background:#0d1117}
  :global(.markdown-body img){max-width:100%;border-radius:6px}
  :global(.markdown-body hr){border:none;border-top:1px solid #30363d;margin:20px 0}
  :global(.markdown-body strong){color:#ffa657}

  .empty-preview { display:flex; align-items:center; justify-content:center; height:100%; color:#484f58; font-style:italic; }

  .editor-textarea::-webkit-scrollbar,.preview-content::-webkit-scrollbar{width:6px}
  .editor-textarea::-webkit-scrollbar-track,.preview-content::-webkit-scrollbar-track{background:#0d1117}
  .editor-textarea::-webkit-scrollbar-thumb,.preview-content::-webkit-scrollbar-thumb{background:#30363d;border-radius:3px}
  .editor-textarea::-webkit-scrollbar-thumb:hover,.preview-content::-webkit-scrollbar-thumb:hover{background:#484f58}
</style>