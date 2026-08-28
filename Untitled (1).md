# Goal: install the Jolli CLI on every platform and verify the install succeeded.

Every step follows the same structure: **Platform → Terminal → Command → Verify**.

---
## **Step 1 — Shell script**
- **Platform:** macOS, Linux, or Windows with WSL
- **Terminal:** [Terminal.app](http://Terminal.app) (macOS) / any Linux shell / WSL terminal
- **Command:**
  ```sh
  curl -fsSL https://jolli.ai/install.sh | sh
  
  ```
- **Verify:**
  ```sh
  jolli --version
  
  ```

  A version number (e.g. `0.99.x`) prints — the CLI is installed and on the PATH. If you see `command not found`, the install failed.
> **Say:** "Okay, I'm on a Mac here. I'll open up Terminal and just paste in this one line. It grabs the install script and runs it — that's it, that's the whole install. Now let's check it worked: `jolli --version`… and there's the version number. We're good. By the way, this exact same line works on Linux and WSL too."
## **Step 2 — PowerShell script**
- **Platform:** Windows 10 / 11
- **Terminal:** PowerShell (Start menu → type "PowerShell")
- **Command:**
  ```powershell
  irm https://jolli.ai/install.ps1 | iex
  
  ```
- **Verify:**
  ```powershell
  jolli --version
  
  ```

  A version number prints — installed. If PowerShell says the term is not recognized, the install failed.
> **Say:** "Alright, now let's do the same thing on Windows. I'll open PowerShell from the Start menu and run this one line — it downloads the installer and runs it. Give it a second… done. Same check as before: `jolli --version`… version number's there, so we're installed on Windows too."
## **Step 3 — CMD batch script**
- **Platform:** Windows 10 / 11 (no PowerShell needed)
- **Terminal:** Command Prompt (Start menu → type "cmd")
- **Command:**
  ```bat
  curl -fsSL https://jolli.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
  
  ```
- **Verify:**
  ```bat
  jolli --version
  
  ```

  A version number prints — installed. `'jolli' is not recognized` means the install failed.
> **Say:** "Now, some Windows machines don't allow PowerShell — no problem, plain old Command Prompt works too. I'll open cmd from the Start menu and run this. It downloads the batch installer, runs it, and cleans up after itself. And again: `jolli --version`… there's the version. Installed."
## **Step 4 — npm**
- **Platform:** any OS with Node.js ≥ 18 already installed
- **Terminal:** any terminal where `npm` works
- **Command:**
  ```sh
  npm install -g @jolli.ai/cli
  
  ```
- **Verify:**
  ```sh
  jolli --version
  
  ```

  A version number prints — installed. You can also confirm with `npm ls -g @jolli.ai/cli`.
> **Say:** "If you've already got Node on your machine, you don't even need the scripts — the CLI is just an npm package. So it's `npm install -g @jolli.ai/cli`, wait for it to finish… and check with `jolli --version`. Version's there, we're done."
## **Step 5 — Homebrew**
- **Platform:** macOS or Linux with Homebrew installed
- **Terminal:** [Terminal.app](http://Terminal.app) / any shell where `brew` works
- **Command:**
  ```sh
  brew install jolli-cli
  
  ```
- **Verify:**
  ```sh
  jolli --version
  
  ```

  A version number prints — installed. `brew list jolli-cli` also shows the installed files.
> **Say:** "And if you're a Homebrew person — yep, it's on Homebrew too. It's in homebrew-core itself, so no extra tap to add, just `brew install jolli-cli` like any other package. Once that's done, `jolli --version`… and there it is."

*Note: pending *[homebrew-core PR #293490](https://github.com/Homebrew/homebrew-core/pull/293490)* — demo this step only after the formula merges.*
## **Step 6 — Editor plugins**
- **Platform:** any OS running VS Code, Cursor / Windsurf, or a JetBrains IDE
- **Terminal:** none — everything happens inside the editor
- **Command:** search **"Jolli Memory"** in the editor's extension marketplace and click **Install**:
  
  | Editor            | Install from                                                                                                           |
  | :----------------- | :---------------------------------------------------------------------------------------------------------------------- |
  | VS Code           | [Marketplace — jolli.jollimemory-vscode](https://marketplace.visualstudio.com/items?itemName=jolli.jollimemory-vscode) |
  | Cursor / Windsurf | [Open VSX — jolli/jollimemory-vscode](https://open-vsx.org/extension/jolli/jollimemory-vscode)                         |
  | JetBrains IDEs    | [JetBrains Marketplace — Jolli Memory](https://plugins.jetbrains.com/search?search=jolli%20memory)                     |
  
- **Verify:** open the editor's installed-extensions list — **Jolli Memory** appears there, enabled.
> **Say:** "Last one — maybe you don't want to touch a terminal at all. That's fine. In VS Code, I open the extension marketplace, search for 'Jolli Memory', hit Install… and to check it worked, I just look at my installed extensions — there it is, enabled. Cursor, Windsurf, and JetBrains all work the same way, just through their own marketplaces."
---
## **Closing line**
> "So that's six ways to get Jolli installed — shell script, PowerShell, Command Prompt, npm, Homebrew, or straight from your editor. Whatever machine you're on, it's one command, and one quick `jolli --version` tells you it worked."
