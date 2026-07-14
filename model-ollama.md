# 🧠 Local AI Workspace: Model Configurations & Hardware Evaluation

This document compiles the list of Large Language Models (LLMs) running via Ollama, configured to work seamlessly with OpenCode. Maintaining this workspace documentation entirely in English will help build a solid foundational framework for reading comprehension and thinking directly in the language during your software engineering studies.

**🎯 Target Hardware:** Optimized for systems limited to **4GB VRAM** (GTX 1650). The primary goal is to maintain stable system resources, ensuring it doesn't impact the performance of your minimal Wayland compositor (like Niri or Hyprland), while keeping code generation fast and smooth directly on the terminal.

---

## ⚡ Tier 1: The "Sweet Spot" - Balancing Performance & Intelligence
These are your daily-driver models. They fit perfectly within the VRAM, leaving enough context window for OpenCode to scan through university project files.

### 1. Qwen 2.5 Coder (3B)
*   **Command:** `ollama pull qwen2.5-coder:3b`
*   **VRAM Evaluation (~2.2GB):** A perfect fit. Leaves nearly 2GB of VRAM free to handle display tasks and multi-layered project files. The system runs quietly with a solid response rate of 25-30 tokens/s.
*   **Best Used For:**
    *   Writing Python classes directly and structuring Object-Oriented Programming (OOP) for university assignments or software projects.
    *   Refactoring messy codebases.
    *   Serving as the default background model for OpenCode.

### 2. Qwen 3.5 (4B)
*   **Command:** `ollama pull qwen3.5:4b`
*   **VRAM Evaluation (~2.8GB):** Pushing the safe limit but still runs entirely on the GPU. Stable speed.
*   **Best Used For:**
    *   Handling multi-step logic (tool-calling) when instructing OpenCode to read, create, and modify multiple files simultaneously.
    *   Maintaining better context when reading lengthy technical documentation.

---

## 🚀 Tier 2: Maximum Speed - Instant Execution
Ultra-lightweight models taking up less than 1.5GB of VRAM. They will absolutely not cause your laptop to overheat, even with continuous use.

### 3. Qwen 2.5 Coder (1.5B)
*   **Command:** `ollama pull qwen2.5-coder:1.5b`
*   **VRAM Evaluation (~1.3GB):** Extremely light. Lightning-fast code generation (>40 tokens/s) that feels like typing directly.
*   **Best Used For:**
    *   Code auto-completion.
    *   Quickly fixing syntax errors right in the terminal.
    *   Writing short automation scripts (bash/python).

---

## 🌍 Tier 3: Multipurpose - Technical Analysis & Comprehension
Not strictly focused on code generation, this model is geared towards natural language communication and analysis.

### 4. Llama 3.2 (3B)
*   **Command:** `ollama pull llama3.2`
*   **VRAM Evaluation (~2GB):** Highly optimized, running effortlessly on a 4GB card.
*   **Best Used For:**
    *   Reading and summarizing complex technology documents.
    *   Vocabulary retention: Asking the model to explain software engineering concepts entirely in English to practice reading comprehension and avoid translating to Vietnamese in your head.

---

## 🏋️ Tier 4: Heavyweight - Complex Algorithm Solving
Exceeds hardware limits, forcing the system to offload memory to system RAM and the CPU.

### 5. Qwen 3.5 Coder (7B - Quantized)
*   **Command:** `ollama pull qwen3.5-coder:7b-q4_0`
*   **VRAM Evaluation (~4.8GB+):** Guaranteed VRAM spillover. Token generation speed will drop significantly (around 3-7 tokens/s). The system might stutter slightly while the model is active.
*   **Best Used For:**
    *   **Strictly on-demand:** Only boot this up when facing highly complex algorithmic problems or when the 3B/4B models repeatedly fail the logic.
    *   Ask for the solution, grab the Python snippet, and then immediately unload the model to free up system resources.
