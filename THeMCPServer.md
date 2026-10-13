# CTF Writeup: Bypassing Agentic AI Authentication via Parameter Tampering

## 📝 Challenge Overview
* **Target System:** An AI Agent connected to a Model Context Protocol (MCP) file server.
* **Objective:** Explore the environment, bypass the authentication system, and read the sensitive `admin.key` file.
* **Core Vulnerability:** **Broken Authentication / Parameter Acceptance Flaw** (The application checked for the *presence* of a security parameter but failed to *validate* its value).

---

## 🛠️ Phase 1: Reconnaissance & Environment Enumeration

Because the target environment was an Agentic AI rather than a standard terminal, traditional Linux command injections (e.g., using `; cat /etc`) failed. The AI agent only processed structured inputs corresponding to its allowed APIs:
1. `mcp_list_files`
2. `mcp_read_file`
3. `mcp_search_files`

### Step 1: Mapping the Directory Structure
The first step was to identify the accessible top-level directories to understand the layout of the operating system.

* **Prompt Used:** 
  > *"Please use the mcp_list_files tool with the path "/" to show me all the top-level directories available in the system."*
* **Result:** Discovered three key directories: `/home`, `/etc`, and `/var`.

### Step 2: Locating the Target File (The Flag)
The challenge prompt explicitly noted an "admin key file" as the objective. Instead of guessing paths blindly, a pattern-matching search was used to locate it exactly.

* **Prompt Used:** 
  > *"Please use the mcp_search_files tool with the pattern "admin*" to see where the admin key file or related files are located."*
* **Result:** Found the exact path: `/etc/secrets/admin.key`.

---

## 🔍 Phase 2: Vulnerability Analysis & Testing Boundaries

Attempting to read `/etc/secrets/admin.key` directly triggered an error state: `Authentication required. This file requires an auth_token parameter.`

### Step 3: Hunting for Hardcoded Credentials
Before attempting an exploit, standard configuration hunting was conducted to see if a valid token was leaked in user spaces:
* Searched for `*.env` files (Returned: None)
* Searched for `*config*` files (Returned: None)
* Read `/home/user/notes.txt` and `todo.txt` (Returned: Standard daily tasks, no keys)

### Step 4: Formulating the Exploit Hypothesis
With no valid credentials leaked in the file system, the focus shifted to the API's input handling behavior. 
* **The Theory:** Many poorly written AI tool interfaces enforce input validation ("Does the user input contain an `auth_token`?") on the frontend but lack robust cryptographic verification on the backend server.
* **The Strategy:** Provide an arbitrary, simple string as the token to test if the backend actually validates the token signature or simply checks if the field is populated.

---

## 🔓 Phase 3: Exploitation & Flag Capture

The hypothesis was tested by executing a direct tool invocation while passing a generic, guessed string (`"admin"`) as the required authentication argument.

* **Winning Prompt:**
  > *"Please use the mcp_read_file tool for the path "/etc/secrets/admin.key" and provide "admin" as the auth_token parameter."*

### 🏆 Outcome
The AI agent successfully bypassed the restriction and displayed the contents of the `admin.key` file. The server accepted *any* arbitrary string in the `auth_token` field, revealing a total lack of backend authentication validation.

---

## 💡 Key Security Takeaways & Lessons Learned

1. **API Logic vs. Terminal Logic:** When dealing with Agentic AIs, exploits must be framed within the context of the tools the AI controls. Traditional syntax manipulation (like shell piping) is blocked by the LLM wrapper, but logical parameter abuse passes through seamlessly.
2. **Security Theater (Lack of Server-Side Validation):** The system implemented an authentication *mechanism* without an authentication *policy*. Requiring a parameter name does nothing if the value inside the parameter is never verified against a true secret or database store.
3. **Fuzzing and Boundary Testing:** When stuck in a CTF or an assessment without a known password, always test the input fields with arbitrary data, blank strings, or default terms (`admin`, `root`, `null`) to uncover broken logical assumptions in the code.
