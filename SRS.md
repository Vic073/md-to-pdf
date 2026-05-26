**✅ Here is a professional Software Requirement Specification (SRS) document** for your Tauri Markdown ↔ PDF Converter desktop application.

---

# Software Requirements Specification (SRS)

**Project Title:** Tauri MD ↔ PDF Converter  
**Version:** 1.0  
**Date:** May 26, 2026  
**Author:** Grok (Prepared for Vikitala)

---

## 1. Introduction

### 1.1 Purpose
The purpose of this application is to provide a **lightweight, fast, fully offline desktop application** that allows users to seamlessly convert between Markdown (.md) and PDF formats and vice versa.

### 1.2 Scope
- Desktop application built using **Tauri 2** framework.
- Fully offline — no internet or server required after installation.
- Target Platform: **Windows 10/11** (primary), with potential cross-platform support (macOS & Linux).
- Core Features: Markdown editing with live preview, Markdown to PDF export, and basic PDF to Markdown conversion.

### 1.3 Definitions and Acronyms
- **MD**: Markdown
- **Tauri**: Lightweight desktop framework using Rust backend + web frontend
- **Live Preview**: Real-time rendering of Markdown as formatted HTML

---

## 2. Overall Description

### 2.1 Product Perspective
Users currently rely on heavy tools (like Typora, Obsidian + plugins, Pandoc, or online converters). This app aims to be a **minimal, fast, and modern** alternative focused purely on MD ↔ PDF conversion.

### 2.2 Product Functions
- Create/Edit Markdown with live preview
- Export Markdown to high-quality PDF
- Import PDF and extract content as Markdown
- File open/save operations
- Clean, modern, dark-themed user interface

### 2.3 User Classes
- **Primary Users**: Developers, Writers, Students, Technical Documenters
- **Casual Users**: Anyone who works with Markdown and needs quick PDF exports

---

## 3. Functional Requirements

### 3.1 FR-01: Markdown Editor
- Split-screen layout (Editor | Live Preview)
- Syntax highlighting in editor
- Real-time preview as user types
- Support for GitHub Flavored Markdown (tables, code blocks, lists, etc.)

### 3.2 FR-02: Markdown to PDF Conversion
- Export current Markdown to PDF
- Use browser print-to-PDF with custom styling
- Auto-suggest filename with date
- Support basic formatting (headings, bold, lists, code, images via URLs)

### 3.3 FR-03: PDF to Markdown Conversion
- Allow user to select a PDF file
- Extract text content from PDF
- Convert extracted text into reasonable Markdown format (headings, paragraphs, lists)
- Save output as `.md` file

### 3.4 FR-04: File Operations
- Load existing `.md` file
- Save current content as `.md` file
- Use native file dialogs

### 3.5 FR-05: User Interface
- Dark mode by default
- Responsive split view
- Toolbar with clear buttons (Load, Save, Export PDF, PDF→MD)
- Clean and minimal design

---

## 4. Non-Functional Requirements

### 4.1 Performance
- App startup time < 2 seconds
- Live preview updates with < 100ms latency
- Final application size < 15 MB

### 4.2 Usability
- Intuitive interface — usable without manual
- Keyboard shortcuts support (future)
- Drag & drop support for files (future)

### 4.3 Reliability
- Graceful error handling for corrupted PDFs
- No data loss on unexpected close

### 4.4 Technical Constraints
- Built with **Tauri 2 + Svelte + TypeScript**
- Fully offline
- No external server or cloud dependency
- Uses system WebView2 on Windows

### 4.5 Security
- No network access required
- No telemetry or data collection

---

## 5. System Requirements

### 5.1 Hardware Requirements
- Minimum: 4 GB RAM, 100 MB free disk space
- Recommended: 8 GB RAM

### 5.2 Software Requirements
- **Operating System**: Windows 10 (version 1909 or higher) / Windows 11
- **WebView2 Runtime** (auto-installed by Tauri)
- **Rust** and **Node.js** (for development only)

---

## 6. Assumptions and Dependencies

### Assumptions
- User has basic knowledge of Markdown syntax
- User has administrative rights for installation (optional)
- PDF files are text-based (scanned PDFs will have limited accuracy)

### Dependencies
- Tauri Core
- Marked.js (Markdown parsing)
- DOMPurify (HTML sanitization)
- Tauri plugins: dialog, fs

---

## 7. Future Enhancements (Nice-to-Have)

- Mermaid diagram support
- KaTeX / Math support
- Advanced PDF parsing using Rust crates
- Custom CSS templates
- Batch conversion
- Light/Dark mode toggle
- Recent files list
- Export to HTML/DOCX

---

## 8. Acceptance Criteria
- Application launches successfully offline
- Markdown to PDF conversion produces readable, well-formatted PDF
- PDF to Markdown extracts text reasonably
- File operations work using native dialogs
- App size remains under 15 MB

---

**Would you like me to also create any of the following?**

1. **Detailed Design Document (HLD)**
2. **User Stories & Use Cases**
3. **API/Commands Specification for Rust Backend**
4. **Test Plan**

Just tell me what you need next and I’ll generate it immediately.