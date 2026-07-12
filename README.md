# DBMS_09 – Desktop Frontend for a REST API

**Module:** Introduction to Database Management Systems · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_09>  
**Prerequisites:** DBMS_01 – DBMS_08, Lecture 10  
**Duration:** 120 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Explore a REST API using its **Swagger UI** at `/docs`
- Explain the difference between a **sequential program** and an **event-driven GUI**
- Separate **data access logic** from **UI code** in a Python desktop application
- Send HTTP requests from a tkinter application using the **`requests`** library
- Display tabular API data in a `ttk.Treeview` widget
- Send write requests (`POST`, `PUT`) triggered by **form inputs and buttons**
- Implement a **modal connection dialog** that collects the API URL and key at startup
- Manage a Python project and its dependencies with **`uv`**
- Package the application as a standalone executable using **PyInstaller**
- Build a **`.deb`** installer (Linux), a **Setup `.exe`** (Windows), and a **`.dmg`** image (macOS)

**After completing this exercise you should be able to answer the following questions independently:**

- Why must the API URL not be hard-coded in the application source?
- What is `root.mainloop()`, and why does a GUI program never reach the line after it during normal operation?
- Why is it bad practice to make HTTP requests directly inside a button callback without threading?
- What does `grab_set()` do in tkinter, and why is it used in a connection dialog?
- What does PyInstaller bundle into the `dist/` directory, and why does the target machine not need Python installed?


### Answers to the introductory questions

**Why must the API URL not be hard-coded in the application source?**  
The API address can change depending on the server, network, or user. Asking for it in the connection dialog allows the same program to connect to different servers without changing the source code.

**What is `root.mainloop()`, and why does a GUI program normally not reach the line after it?**  
`root.mainloop()` starts tkinter's event loop. It keeps waiting for events such as button clicks, keyboard input, and window actions. During normal operation, the loop continues until the main window is closed, so the next line is not executed before that.

**Why is it bad practice to make HTTP requests directly inside a button callback without threading?**  
Tkinter normally runs the interface in one main thread. A slow HTTP request blocks that thread, which can freeze the window and make it appear unresponsive. The request should run in a background thread, while GUI updates are sent back to the main thread with `after()`.

**What does `grab_set()` do in tkinter, and why is it used in a connection dialog?**  
`grab_set()` makes the dialog modal. The user cannot interact with the main window until the dialog is closed. This ensures that the API URL and key are confirmed before the rest of the application is used.

**What does PyInstaller bundle into the `dist/` directory, and why does the target machine not need Python installed?**  
PyInstaller bundles the application code, a Python interpreter, imported libraries, and supporting files. Since the interpreter and dependencies are included, the target machine does not need a separate Python installation.

---

## Background

In lecture 10 we built a tkinter frontend for a student database API. In this exercise you will apply the same patterns to the **Factory Demo API** — a material planning system for a bicycle frame production facility.

The API is already deployed on the lecture server at **`http://128.140.85.215:8888`**. You can explore every available endpoint interactively in the Swagger UI at:

```
http://128.140.85.215:8888/docs
```

Open that URL in your browser before writing a single line of code — the Swagger UI lets you read the request/response schema for each endpoint and try them out directly.

> **API key:** The read endpoints are public, but the write endpoints (`POST` / `PUT`) require the header `X-API-Key`. **You find the current API key in our Discord channel** — it is deliberately not published in this repository.

> **Screenshot 1:** Take a screenshot of the Swagger UI showing the list of available endpoints.
>
> <img width="1255" height="952" alt="1" src="https://github.com/user-attachments/assets/6e3c5513-b458-4922-a4f9-79d57384e9ae" />

---

## API Overview

The following endpoints are available. All read endpoints are public; write endpoints require the header `X-API-Key`.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/teile` | All parts with stock and reorder flag |
| `GET` | `/produkte` | All products with stock and running totals |
| `GET` | `/stueckliste/{produkt_id}` | Bill of materials for a product |
| `GET` | `/bestellwarnungen` | Reorder warning log |
| `POST` | `/wareneingang` | Deliver parts to stock |
| `POST` | `/produktion` | Record a production run |
| `POST` | `/lagerausgang` | Check out finished products |
| `PUT` | `/teile/{id}` | Manually set part stock (stocktake) |

Study the request body and response schemas in the Swagger UI before building the client code in Section 3.

---

## Prerequisites Check

### Step 1 – Verify uv is installed

```bash
uv --version
```

If `uv` is not installed:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

On Windows (PowerShell):

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Step 2 – Verify Python 3.11+

```bash
python3 --version
```

> Both commands should succeed. If Python is missing, `uv` can install it: `uv python install 3.12`

> **Screenshot 2:** Take a screenshot showing the version outputs of `uv` and `python3`.
>
> <img width="386" height="179" alt="2" src="https://github.com/user-attachments/assets/9c3acfc0-7ee7-4ab8-ae00-b7f7a6ae569f" />


---

## 1 – Project Setup with uv

### Step 1 – Initialise the Project

```bash
uv init fabrik-frontend
cd fabrik-frontend
git init
git remote add origin git@github.com:<your-username>/fabrik-frontend.git
```

`uv init` creates this structure:

```
fabrik-frontend/
  pyproject.toml
  src/
    fabrik_frontend/
      __init__.py
  .python-version
  README.md
```

### Step 2 – Add Dependencies

```bash
uv add requests
```

`uv add` writes the dependency into `pyproject.toml`, updates `uv.lock`, and installs the package. Commit `uv.lock` — it pins exact versions for reproducible builds.

### Step 3 – Verify pyproject.toml

```bash
cat pyproject.toml
```

It should look like this:

```toml
[project]
name = "fabrik-frontend"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["requests>=2.32"]
```

### Step 4 – Run the Project (Smoke Test)

```bash
uv run python -c "import requests; print('requests', requests.__version__)"
```

> **Screenshot 3:** Take a screenshot showing the output confirming `requests` is available.
>
> <img width="959" height="68" alt="3" src="https://github.com/user-attachments/assets/c721b7d0-ad2f-420b-9a17-4d549a591bf7" />


### Step 5 – Commit

```bash
git add pyproject.toml uv.lock src/
git commit -m "feat: initialise project with uv"
git push -u origin main
```

### Questions for Section 1

**Question 1.1:** `uv` creates a `uv.lock` file alongside `pyproject.toml`. What is the difference between the two files? Why should `uv.lock` be committed to version control?

> **Answer:** `pyproject.toml` describes the project and lists the dependencies it is allowed to use, for example `requests>=2.32`. `uv.lock` records the exact versions that were resolved, including indirect dependencies such as `urllib3` and `certifi`. The lock file should be committed so that other developers and build systems install the same versions and get reproducible results.

**Question 1.2:** `uv run` executes a command inside the project's virtual environment without you having to activate it manually. What problem does this solve compared to relying on the system-wide Python installation?

> **Answer:** `uv run` automatically uses the project's virtual environment. This prevents the application from accidentally using a different system Python or incompatible globally installed packages. It also means that the command works without manually activating the virtual environment first.

---

## 2 – API Client Layer

The API client is a separate module (`api.py`) that handles all HTTP communication. The UI code never calls `requests` directly — it calls functions from `api.py`. This separation makes both layers independently testable and readable.

### Step 1 – Create api.py

```bash
touch src/fabrik_frontend/api.py
```

```python
import requests

BASE_URL = "http://localhost:8888"
HEADERS = {}


# --- read endpoints (no key required) ---

def get_teile():
    r = requests.get(f"{BASE_URL}/teile", timeout=5)
    r.raise_for_status()
    return r.json()

def get_produkte():
    r = requests.get(f"{BASE_URL}/produkte", timeout=5)
    r.raise_for_status()
    return r.json()

def get_stueckliste(produkt_id: int):
    r = requests.get(f"{BASE_URL}/stueckliste/{produkt_id}", timeout=5)
    r.raise_for_status()
    return r.json()

def get_bestellwarnungen():
    r = requests.get(f"{BASE_URL}/bestellwarnungen", timeout=5)
    r.raise_for_status()
    return r.json()


# --- write endpoints (API key required) ---

def post_wareneingang(teil_id: int, menge: int, notiz: str = ""):
    payload = {"teil_id": teil_id, "menge": menge, "notiz": notiz or None}
    r = requests.post(f"{BASE_URL}/wareneingang", json=payload, headers=HEADERS, timeout=5)
    r.raise_for_status()
    return r.json()

def post_produktion(produkt_id: int, menge: int):
    payload = {"produkt_id": produkt_id, "menge": menge}
    r = requests.post(f"{BASE_URL}/produktion", json=payload, headers=HEADERS, timeout=5)
    r.raise_for_status()
    return r.json()

def post_lagerausgang(produkt_id: int, menge: int, notiz: str = ""):
    payload = {"produkt_id": produkt_id, "menge": menge, "notiz": notiz or None}
    r = requests.post(f"{BASE_URL}/lagerausgang", json=payload, headers=HEADERS, timeout=5)
    r.raise_for_status()
    return r.json()

def put_teil_bestand(teil_id: int, bestand: int):
    payload = {"bestand": bestand}
    r = requests.put(f"{BASE_URL}/teile/{teil_id}", json=payload, headers=HEADERS, timeout=5)
    r.raise_for_status()
    return r.json()
```

### Step 2 – Test the Client Manually

Open an interactive Python session in the project environment:

```bash
uv run python
```

```python
from fabrik_frontend import api
api.BASE_URL = "<base-url-from-instructor>"

parts = api.get_teile()
for p in parts:
    print(p["name"], p["bestand"])
```

> **Screenshot 4:** Take a screenshot showing the parts list printed in the Python REPL.
>
> <img width="472" height="246" alt="4" src="https://github.com/user-attachments/assets/b48a31e1-88c5-4a19-a3da-c477aa1faadc" />


Exit with `exit()`.

### Step 3 – Commit

```bash
git add src/fabrik_frontend/api.py
git commit -m "feat: add API client layer for all endpoints"
git push
```

### Questions for Section 2

**Question 2.1:** `r.raise_for_status()` raises an exception if the server returned a 4xx or 5xx status code. What would happen if this call were omitted and the server returned `409 Conflict`?

> **Answer:** Without `r.raise_for_status()`, the program would not automatically treat `409 Conflict` as an error. It might continue, return the server's error JSON as if it were a successful result, or fail later when expected fields are missing. Calling `raise_for_status()` stops the normal flow immediately and lets the GUI show an error message.

**Question 2.2:** `BASE_URL` and `HEADERS` are module-level variables set at runtime by the connection dialog. Why is this approach preferable to reading them from a configuration file on disk?

> **Answer:** Setting `BASE_URL` and `HEADERS` at runtime makes it easy to connect to different servers without editing the source code. The API key also remains only in memory for the current session instead of being stored permanently in a file. This is simple and reduces the risk of accidentally committing the key to Git, although a dedicated secure credential store would be preferable for a production application.

---

## 3 – Connection Dialog

The API URL and key must not be hard-coded. A modal dialog collects them at startup.

### Step 1 – Create connection_dialog.py

```bash
touch src/fabrik_frontend/connection_dialog.py
```

```python
import tkinter as tk
from tkinter import messagebox


class ConnectionDialog(tk.Toplevel):
    def __init__(self, parent: tk.Tk):
        super().__init__(parent)
        self.title("Connect to API")
        self.resizable(False, False)
        self.grab_set()
        self.confirmed = False

        self._url   = tk.StringVar(value="http://localhost:8888")
        self._token = tk.StringVar()

        tk.Label(self, text="API URL:").grid(
            row=0, column=0, padx=14, pady=10, sticky=tk.E)
        tk.Entry(self, textvariable=self._url, width=40).grid(
            row=0, column=1, padx=10)

        tk.Label(self, text="X-API-Key:").grid(
            row=1, column=0, padx=14, pady=10, sticky=tk.E)
        tk.Entry(self, textvariable=self._token,
                 width=40, show="*").grid(row=1, column=1, padx=10)

        tk.Button(self, text="Connect",
                  command=self._confirm).grid(
                      row=2, column=0, columnspan=2, pady=12)

        self.protocol("WM_DELETE_WINDOW", self._cancel)
        self.wait_window()

    def _confirm(self):
        if not self._url.get().startswith("http"):
            messagebox.showwarning("Input error", "Please enter a valid URL.")
            return
        self.confirmed = True
        self.destroy()

    def _cancel(self):
        self.destroy()

    @property
    def url(self) -> str:
        return self._url.get().rstrip("/")

    @property
    def token(self) -> str:
        return self._token.get()
```

### Step 2 – Explain grab_set()

Run the following to see what a non-modal dialog looks like — and why `grab_set()` matters:

```python
import tkinter as tk

root = tk.Tk()
root.title("Main Window")

top = tk.Toplevel(root)
top.title("Dialog without grab_set")
# Without grab_set() the user can click the main window while the dialog is open.

root.mainloop()
```

> **Question:** Click the main window while the dialog is open. What happens?
> Now add `top.grab_set()` after the `Toplevel` creation and repeat.
> Describe the difference.

> **Answer:** Without `grab_set()`, I could still click and interact with the main window while the dialog was open. After adding `grab_set()`, the main window no longer accepted input until the dialog was closed. This makes the connection dialog modal and ensures that the user finishes the connection step first.

### Step 3 – Commit

```bash
git add src/fabrik_frontend/connection_dialog.py
git commit -m "feat: add modal connection dialog for URL and API key"
git push
```

---

## 4 – Main Window: Parts Tab

The main window uses a `ttk.Notebook` to organise four tabs. Build them one at a time.

### Step 1 – Create ui.py with the Parts Tab

```bash
touch src/fabrik_frontend/ui.py
```

```python
import tkinter as tk
from tkinter import ttk, messagebox
from fabrik_frontend import api


class App(tk.Frame):
    def __init__(self, master: tk.Tk):
        super().__init__(master)
        master.title("Factory Demo – Material Planning")
        master.minsize(820, 520)
        self.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        notebook = ttk.Notebook(self)
        notebook.pack(fill=tk.BOTH, expand=True)

        self._build_parts_tab(notebook)
        self._build_products_tab(notebook)
        self._build_bom_tab(notebook)
        self._build_warnings_tab(notebook)

        self._refresh_all()

    # ------------------------------------------------------------------
    # Parts tab
    # ------------------------------------------------------------------

    def _build_parts_tab(self, notebook):
        frame = ttk.Frame(notebook)
        notebook.add(frame, text="Parts")

        # Treeview
        cols = ("id", "name", "unit", "stock", "min_stock", "reorder")
        self._parts_tree = ttk.Treeview(frame, columns=cols, show="headings", height=10)
        for col, heading, width in [
            ("id",       "ID",             40),
            ("name",     "Part",          180),
            ("unit",     "Unit",           60),
            ("stock",    "Stock",          70),
            ("min_stock","Min stock",      70),
            ("reorder",  "Reorder?",       70),
        ]:
            self._parts_tree.heading(col, text=heading)
            self._parts_tree.column(col, width=width, anchor=tk.CENTER)
        self._parts_tree.pack(fill=tk.BOTH, expand=True, pady=(0, 8))

        btn_row = ttk.Frame(frame)
        btn_row.pack(fill=tk.X)
        ttk.Button(btn_row, text="Refresh",        command=self._load_parts).pack(side=tk.LEFT, padx=4)

        # Deliver parts form
        delivery = ttk.LabelFrame(frame, text=" Deliver parts (POST /wareneingang) ")
        delivery.pack(fill=tk.X, pady=8)
        self._d_teil_id = self._labeled_entry(delivery, "Part ID", 0)
        self._d_menge   = self._labeled_entry(delivery, "Quantity", 1)
        self._d_notiz   = self._labeled_entry(delivery, "Note (optional)", 2)
        ttk.Button(delivery, text="Deliver",
                   command=self._deliver).grid(row=3, column=1, pady=6, sticky=tk.W)

        # Stocktake form
        stocktake = ttk.LabelFrame(frame, text=" Set stock manually – stocktake (PUT /teile/{id}) ")
        stocktake.pack(fill=tk.X, pady=4)
        self._s_teil_id = self._labeled_entry(stocktake, "Part ID", 0)
        self._s_bestand = self._labeled_entry(stocktake, "New stock", 1)
        ttk.Button(stocktake, text="Set stock",
                   command=self._set_stock).grid(row=2, column=1, pady=6, sticky=tk.W)

    def _load_parts(self):
        for row in self._parts_tree.get_children():
            self._parts_tree.delete(row)
        try:
            for p in api.get_teile():
                self._parts_tree.insert("", tk.END, values=(
                    p["id"], p["name"], p["einheit"],
                    p["bestand"], p["mindestbestand"],
                    "YES" if p["unter_mindestbestand"] else "no",
                ))
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def _deliver(self):
        try:
            result = api.post_wareneingang(
                int(self._d_teil_id.get()),
                int(self._d_menge.get()),
                self._d_notiz.get(),
            )
            messagebox.showinfo("Delivered", str(result))
            self._load_parts()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def _set_stock(self):
        try:
            result = api.put_teil_bestand(
                int(self._s_teil_id.get()),
                int(self._s_bestand.get()),
            )
            messagebox.showinfo("Updated", str(result))
            self._load_parts()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    # ------------------------------------------------------------------
    # Products tab
    # ------------------------------------------------------------------

    def _build_products_tab(self, notebook):
        frame = ttk.Frame(notebook)
        notebook.add(frame, text="Products")

        cols = ("id", "name", "stock", "total_produced", "total_checked_out")
        self._prod_tree = ttk.Treeview(frame, columns=cols, show="headings", height=8)
        for col, heading, width in [
            ("id",                "ID",             40),
            ("name",              "Product",       200),
            ("stock",             "In stock",       80),
            ("total_produced",    "Total produced", 100),
            ("total_checked_out", "Total out",      80),
        ]:
            self._prod_tree.heading(col, text=heading)
            self._prod_tree.column(col, width=width, anchor=tk.CENTER)
        self._prod_tree.pack(fill=tk.BOTH, expand=True, pady=(0, 8))

        ttk.Button(frame, text="Refresh", command=self._load_products).pack(anchor=tk.W)

        prod_form = ttk.LabelFrame(frame, text=" Record production run (POST /produktion) ")
        prod_form.pack(fill=tk.X, pady=8)
        self._p_prod_id = self._labeled_entry(prod_form, "Product ID", 0)
        self._p_menge   = self._labeled_entry(prod_form, "Quantity", 1)
        ttk.Button(prod_form, text="Produce",
                   command=self._produce).grid(row=2, column=1, pady=6, sticky=tk.W)

        out_form = ttk.LabelFrame(frame, text=" Check out from warehouse (POST /lagerausgang) ")
        out_form.pack(fill=tk.X, pady=4)
        self._o_prod_id = self._labeled_entry(out_form, "Product ID", 0)
        self._o_menge   = self._labeled_entry(out_form, "Quantity", 1)
        self._o_notiz   = self._labeled_entry(out_form, "Note (optional)", 2)
        ttk.Button(out_form, text="Check out",
                   command=self._checkout).grid(row=3, column=1, pady=6, sticky=tk.W)

    def _load_products(self):
        for row in self._prod_tree.get_children():
            self._prod_tree.delete(row)
        try:
            for p in api.get_produkte():
                self._prod_tree.insert("", tk.END, values=(
                    p["id"], p["name"], p["bestand"],
                    p["gesamt_produziert"], p["gesamt_ausgecheckt"],
                ))
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def _produce(self):
        try:
            result = api.post_produktion(
                int(self._p_prod_id.get()), int(self._p_menge.get()))
            messagebox.showinfo("Produced", str(result))
            self._load_products()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def _checkout(self):
        try:
            result = api.post_lagerausgang(
                int(self._o_prod_id.get()),
                int(self._o_menge.get()),
                self._o_notiz.get(),
            )
            messagebox.showinfo("Checked out", str(result))
            self._load_products()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    # ------------------------------------------------------------------
    # Bill of materials tab
    # ------------------------------------------------------------------

    def _build_bom_tab(self, notebook):
        frame = ttk.Frame(notebook)
        notebook.add(frame, text="Bill of Materials")

        input_row = ttk.Frame(frame)
        input_row.pack(fill=tk.X, pady=6)
        tk.Label(input_row, text="Product ID:").pack(side=tk.LEFT, padx=4)
        self._bom_id = ttk.Entry(input_row, width=6)
        self._bom_id.pack(side=tk.LEFT)
        ttk.Button(input_row, text="Load",
                   command=self._load_bom).pack(side=tk.LEFT, padx=6)

        cols = ("teil_id", "name", "qty", "unit")
        self._bom_tree = ttk.Treeview(frame, columns=cols, show="headings", height=12)
        for col, heading, width in [
            ("teil_id", "Part ID",  60),
            ("name",    "Part",    220),
            ("qty",     "Qty",      60),
            ("unit",    "Unit",     80),
        ]:
            self._bom_tree.heading(col, text=heading)
            self._bom_tree.column(col, width=width, anchor=tk.CENTER)
        self._bom_tree.pack(fill=tk.BOTH, expand=True)

    def _load_bom(self):
        for row in self._bom_tree.get_children():
            self._bom_tree.delete(row)
        try:
            for pos in api.get_stueckliste(int(self._bom_id.get())):
                self._bom_tree.insert("", tk.END, values=(
                    pos["teil_id"], pos["teil_name"],
                    pos["menge"], pos["einheit"],
                ))
        except Exception as e:
            messagebox.showerror("Error", str(e))

    # ------------------------------------------------------------------
    # Reorder warnings tab
    # ------------------------------------------------------------------

    def _build_warnings_tab(self, notebook):
        frame = ttk.Frame(notebook)
        notebook.add(frame, text="Reorder Warnings")

        ttk.Button(frame, text="Refresh",
                   command=self._load_warnings).pack(anchor=tk.W, pady=6)

        cols = ("id", "teil_id", "name", "stock_at_warning", "timestamp")
        self._warn_tree = ttk.Treeview(frame, columns=cols, show="headings", height=14)
        for col, heading, width in [
            ("id",               "ID",          40),
            ("teil_id",          "Part ID",     60),
            ("name",             "Part",       180),
            ("stock_at_warning", "Stock",       70),
            ("timestamp",        "Timestamp",  180),
        ]:
            self._warn_tree.heading(col, text=heading)
            self._warn_tree.column(col, width=width, anchor=tk.CENTER)
        self._warn_tree.pack(fill=tk.BOTH, expand=True)

    def _load_warnings(self):
        for row in self._warn_tree.get_children():
            self._warn_tree.delete(row)
        try:
            for w in api.get_bestellwarnungen():
                self._warn_tree.insert("", tk.END, values=(
                    w["id"], w["teil_id"], w["teil_name"],
                    w["bestand_bei_warnung"], w["zeitstempel"],
                ))
        except Exception as e:
            messagebox.showerror("Error", str(e))

    # ------------------------------------------------------------------
    # Helpers
    # ------------------------------------------------------------------

    def _labeled_entry(self, parent, label: str, row: int) -> ttk.Entry:
        ttk.Label(parent, text=label + ":").grid(
            row=row, column=0, padx=8, pady=4, sticky=tk.E)
        entry = ttk.Entry(parent, width=30)
        entry.grid(row=row, column=1, padx=8, pady=4, sticky=tk.W)
        return entry

    def _refresh_all(self):
        self._load_parts()
        self._load_products()
        self._load_warnings()
```

### Step 2 – Create \_\_main\_\_.py

```bash
touch src/fabrik_frontend/__main__.py
```

```python
import tkinter as tk
from fabrik_frontend import api
from fabrik_frontend.connection_dialog import ConnectionDialog
from fabrik_frontend.ui import App


def main():
    root = tk.Tk()
    root.withdraw()

    dialog = ConnectionDialog(root)
    if not dialog.confirmed:
        root.destroy()
        return

    api.BASE_URL = dialog.url
    api.HEADERS  = {"X-API-Key": dialog.token}

    root.deiconify()
    App(root).mainloop()


if __name__ == "__main__":
    main()
```

### Step 3 – Run the Application

```bash
uv run python -m fabrik_frontend
```

The connection dialog should appear. Enter the base URL (`http://128.140.85.215:8888`) and the API key from our Discord channel.

> **Screenshot 5:** Take a screenshot of the connection dialog.
>
> <img width="505" height="197" alt="5" src="https://github.com/user-attachments/assets/162f5518-7d34-4486-a088-795efceef00b" />


> **Screenshot 6:** Take a screenshot of the main window showing the Parts tab populated with data from the live server.
>
> <img width="829" height="642" alt="6" src="https://github.com/user-attachments/assets/0af33ae6-61af-4d45-8732-31bec3c9f019" />

### Step 4 – Commit

```bash
git add src/fabrik_frontend/ui.py src/fabrik_frontend/__main__.py
git commit -m "feat: complete tkinter UI with four tabs and connection dialog"
git push
```

### Questions for Section 4

**Question 4.1:** The `_refresh_all()` method is called in `__init__` and makes three HTTP requests before `mainloop()` starts. In what scenario could this block the UI from appearing? How would you fix it?

> **Answer:** If the server is offline, slow, or unreachable, the three HTTP requests can take several seconds because each one waits for a response or timeout. Since this happens before `mainloop()` starts, the main window may appear late or look frozen. A simple improvement is to schedule the refresh after startup with `root.after(...)`. A better solution is to perform the HTTP requests in a background thread and update the widgets safely through `after()`.

**Question 4.2:** When `api.post_produktion()` raises an exception (e.g. `409 Conflict` due to insufficient parts), `messagebox.showerror` displays the error to the user. Look at the `requests` library documentation: what type of exception does `raise_for_status()` raise, and what attribute contains the server's response body?

> **Answer:** `raise_for_status()` raises `requests.exceptions.HTTPError`. The exception's `response` attribute contains the server response. The response body can be read with `e.response.text`, or with `e.response.json()` when the server returned JSON.

---

## 5 – Walk Through All Endpoints

Work through each operation below in your running application. For each one, verify the result by switching to the relevant tab and pressing **Refresh**, or by checking `/docs` on the server.

### Step 1 – Deliver Parts

In the **Parts** tab, use the *Deliver parts* form:

- Part ID: `1` (Steel tube), Quantity: `20`
- Part ID: `2` (Welding wire), Quantity: `40`

> **Screenshot 7:** Parts tab after delivering, showing updated stock values.
>
> <img width="834" height="637" alt="7" src="https://github.com/user-attachments/assets/e457ffe6-bcf9-461b-9e7e-5949adce9d02" />


### Step 2 – Record a Production Run

In the **Products** tab, use the *Record production run* form:

- Product ID: `1` (City bike frame), Quantity: `3`

> After submitting, switch to the **Parts** tab and refresh. Confirm that steel tubes decreased by 9 (3 × 3) and welding wire by 15 (3 × 5).

> **Screenshot 8:** Parts tab after production, showing reduced stock.
>
> <img width="854" height="646" alt="8" src="https://github.com/user-attachments/assets/6c7cfe3f-b0bd-4385-b616-f447fa2cb5fa" />

### Step 3 – Check the Bill of Materials

Switch to the **Bill of Materials** tab. Enter Product ID `2` (Racing bike frame) and press **Load**.

> **Screenshot 9:** Bill of Materials tab showing all five parts for the racing bike frame.
>
> <img width="843" height="649" alt="9" src="https://github.com/user-attachments/assets/817b5cef-5421-4ff5-9889-048eee0c65ee" />


### Step 4 – Check Out a Finished Product

In the **Products** tab, use the *Check out from warehouse* form:

- Product ID: `1`, Quantity: `1`, Note: `Test order`

> **Screenshot 10:** Products tab after checkout, showing `Total out` incremented by 1.
>
> <img width="827" height="636" alt="10" src="https://github.com/user-attachments/assets/30b3a277-5231-4de8-b861-03eb80ea2415" />


### Step 5 – Trigger a Reorder Warning

Deliver only 1 steel tube (Part ID `1`) to bring stock very low, then produce 1 more frame to push it below the reorder threshold. Switch to the **Reorder Warnings** tab and press **Refresh**.

> **Screenshot 11:** Reorder Warnings tab showing at least one warning entry.
>
> <img width="794" height="80" alt="11" src="https://github.com/user-attachments/assets/5054e2df-efb9-4c79-be3e-44f0cf37d063" />


### Step 6 – Stocktake Correction

In the **Parts** tab, use the *Set stock manually* form to restore a realistic stock level for steel tubes (e.g. `45`).

---

## 6 – Packaging

### Step 1 – Add Entry Point to pyproject.toml

Edit `pyproject.toml`:

```toml
[project]
name = "fabrik-frontend"
version = "0.1.0"
description = "Desktop frontend for the Factory Demo REST API"
requires-python = ">=3.11"
dependencies = ["requests>=2.32"]

[project.scripts]
fabrik-frontend = "fabrik_frontend.__main__:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

Install `hatchling` as a development dependency:

```bash
uv add --dev hatchling
```

Build a wheel and source distribution:

```bash
uv build
ls dist/
```

Expected output:

```
fabrik_frontend-0.1.0-py3-none-any.whl
fabrik_frontend-0.1.0.tar.gz
```

The wheel can be installed on any machine that has Python 3.11+:

```bash
pip install dist/fabrik_frontend-0.1.0-py3-none-any.whl
fabrik-frontend
```

> **Screenshot 12:** Terminal showing `ls dist/` with both distribution files.
>
> <img width="654" height="178" alt="12" src="https://github.com/user-attachments/assets/fa2f0bd8-8e38-4913-a6c2-851e0c84b3a4" />


### Step 2 – Commit

```bash
echo "dist/" >> .gitignore
echo "build/" >> .gitignore
git add pyproject.toml uv.lock .gitignore
git commit -m "feat: add entry point and build config to pyproject.toml"
git push
```

### Questions for Section 6

**Question 6.1:** A `.whl` file still requires Python to be installed on the target machine. What problem does PyInstaller solve that `uv build` does not?

> **Answer:** A wheel contains the Python package but still needs a compatible Python interpreter and its dependencies on the target machine. PyInstaller bundles the interpreter, the dependencies, and the application files into a standalone distribution. Therefore, the user can run the program without installing Python separately.

**Question 6.2:** `[project.scripts]` defines `fabrik-frontend = "fabrik_frontend.__main__:main"`. Explain what happens when a user runs the command `fabrik-frontend` in their terminal after installing the wheel.

> **Answer:** After the wheel is installed, the packaging system creates a command named `fabrik-frontend`. Running that command imports `fabrik_frontend.__main__` and calls its `main()` function. The `main()` function then creates the tkinter root window, opens the connection dialog, configures the API client, and starts the application.

---

## 7 – Standalone Executable with PyInstaller

PyInstaller bundles the Python interpreter, all dependencies, and your application into a self-contained directory. The user does not need Python installed.

### Step 1 – Install PyInstaller

```bash
uv add --dev pyinstaller
```

### Step 2 – Build

```bash
uv run pyinstaller \
  --name fabrik-frontend \
  --onedir \
  --windowed \
  src/fabrik_frontend/__main__.py
```

> `--windowed` suppresses the terminal window on Windows and macOS. On Linux it has no effect.

Inspect the output:

```bash
ls dist/fabrik-frontend/
```

Expected (Linux/macOS):

```
fabrik-frontend     ← executable
_internal/          ← Python interpreter and all libraries
```

On Windows: `fabrik-frontend.exe` instead.

### Step 3 – Run the Executable

```bash
./dist/fabrik-frontend/fabrik-frontend
```

> The application should open exactly as when launched with `uv run python -m fabrik_frontend`.

> **Screenshot 13:** The application running from the PyInstaller-built executable.
>
> <img width="826" height="248" alt="13" src="https://github.com/user-attachments/assets/346eebc2-d489-425f-a5d4-3e920e2b6581" />


> **Note:** PyInstaller builds are platform-specific. A build on Linux produces a Linux binary only. To distribute for all three platforms you need to build once on each operating system (or use a CI/CD pipeline).

---

## 8 – Native Installers

### Linux – .deb Package

Install `fpm` (Effing Package Management):

```bash
sudo apt install ruby ruby-dev build-essential
sudo gem install fpm
```

Create the package directory structure and copy the build:

```bash
mkdir -p pkg/usr/bin
cp -r dist/fabrik-frontend pkg/usr/bin/
```

Build the `.deb`:

```bash
fpm -s dir -t deb \
  --name fabrik-frontend \
  --version 0.1.0 \
  --architecture amd64 \
  --description "Factory Demo desktop frontend" \
  -C pkg .
```

Install and test:

```bash
sudo dpkg -i fabrik-frontend_0.1.0_amd64.deb
fabrik-frontend
```

> **Screenshot 14:** Terminal showing the `.deb` installation and the application launching from `/usr/bin/fabrik-frontend`.
>
> <img width="796" height="338" alt="14" src="https://github.com/user-attachments/assets/63b663bb-6f7e-44dc-bd1e-c0ff243e2797" />


---

### Windows – Setup Installer with Inno Setup

Perform this section on a Windows machine after running PyInstaller there.

Download and install **Inno Setup** from <https://jrsoftware.org/isinfo.php>.

Create `installer.iss` in the project root:

```ini
[Setup]
AppName=Fabrik Frontend
AppVersion=0.1.0
DefaultDirName={autopf}\FabrikFrontend
OutputBaseFilename=fabrik-frontend-setup-0.1.0
Compression=lzma
SolidCompression=yes

[Files]
Source: "dist\fabrik-frontend\*"; DestDir: "{app}"; \
  Flags: ignoreversion recursesubdirs

[Icons]
Name: "{group}\Fabrik Frontend"; Filename: "{app}\fabrik-frontend.exe"
Name: "{commondesktop}\Fabrik Frontend"; Filename: "{app}\fabrik-frontend.exe"
```

Compile the installer:

```bash
iscc installer.iss
```

Result: `fabrik-frontend-setup-0.1.0.exe` — a standard Windows installation wizard.

> **Screenshot 15 (Windows only):** The Inno Setup installer wizard running.
>
> `[insert screenshot]`

---

### macOS – .dmg Image

Perform this section on a macOS machine after running PyInstaller there.

Install `create-dmg`:

```bash
brew install create-dmg
```

PyInstaller produces a `.app` bundle with `--windowed` on macOS. Create the disk image:

```bash
create-dmg \
  --volname "Fabrik Frontend" \
  --icon-size 128 \
  --icon "fabrik-frontend.app" 140 190 \
  --app-drop-link 400 190 \
  "fabrik-frontend-0.1.0.dmg" \
  "dist/"
```

The `.dmg` opens as a drag-and-drop window: the user drags `fabrik-frontend.app` into the Applications folder.

> **Screenshot 16 (macOS only):** The `.dmg` drag-and-drop window.
>
> `[insert screenshot]`

---

### Step – Commit Installer Files

```bash
echo "pkg/" >> .gitignore
git add installer.iss .gitignore
git commit -m "feat: add Inno Setup script for Windows installer"
git push
```

### Questions for Section 8

**Question 8.1:** PyInstaller bundles a complete Python interpreter into `_internal/`. What is the typical size of a PyInstaller `--onedir` output compared to a minimal Python installation, and why is `--onedir` generally preferred over `--onefile` for desktop applications?

> **Answer:** A PyInstaller `--onedir` build is usually tens of megabytes and can exceed 100 MB, depending on the Python version and included libraries. It is often similar in size to, or larger than, a minimal Python installation because it contains the interpreter and all required dependencies. `--onedir` is generally preferred for desktop applications because it starts faster, does not extract all files on every launch, is easier to debug, and is often less likely to trigger antivirus warnings than `--onefile`.

**Question 8.2:** A `.deb` package installed via `dpkg -i` does not appear in the system package manager's update mechanism. Which tool and repository format would you use to distribute updates automatically to Debian/Ubuntu users?

> **Answer:** I would distribute the package through an APT repository. An APT repository stores `.deb` files together with repository metadata such as `Packages`, `Release`, and signed `InRelease` files. Tools such as `aptly` or `reprepro` can create and manage this repository. Users can then receive updates through `apt update` and `apt upgrade`.

---

## 9 – Reflection

**Question A – Separation of Concerns:**  
`api.py` contains all HTTP logic; `ui.py` contains all widget code; `__main__.py` wires them together. Name one concrete benefit this separation provides when you want to write automated tests for the API client.

> **Answer:** The API client can be tested without starting tkinter. For example, a test can mock `requests.get()` or `requests.post()` and verify that `api.py` sends the correct URL, payload, and headers, and that it handles server errors correctly. No windows or button clicks are needed.

**Question B – Event-Driven vs Sequential:**  
A fellow student proposes using a `while True` loop in the main thread to poll the API every 5 seconds and update the display. Explain why this approach would break the tkinter application, and describe the correct alternative.

> **Answer:** A `while True` loop in the main thread would block tkinter's event loop. The window would stop repainting, buttons would not respond, and the application would appear frozen. The correct alternative is to schedule repeated work with `root.after(5000, refresh_function)`. If the HTTP call itself may be slow, it should run in a background thread and the GUI should be updated through `after()`.

**Question C – API Key in the Dialog:**  
The connection dialog collects the API key at runtime and stores it in `api.HEADERS` for the session only. It is never written to disk. What are the security advantages of this approach compared to storing the key in a configuration file in the user's home directory?

> **Answer:** Keeping the API key only in memory means it is not permanently stored in a readable file, cannot easily be copied from the user's home directory, and is less likely to be committed to Git or included in a backup. It also disappears when the application closes. This reduces exposure, although the key is still present in process memory while the application is running.

**Question D – The Full Stack:**  
You have now touched every layer of the system: PostgreSQL database → Docker Compose deployment → FastAPI REST layer → tkinter desktop client → native installer. Describe in one sentence the role of each layer, and explain which layer a new employee would need to understand to add a sixth part to the bill of materials without changing any other layer.

> **Answer:** PostgreSQL stores the persistent data; Docker Compose starts and connects the required services; FastAPI exposes the data and business rules through REST endpoints; tkinter provides the desktop user interface; and the native installer installs and launches the application on the target operating system. To add a sixth part to a bill of materials without changing the other layers, a new part and its product-to-part relationship should be added in the database, because the API and GUI already display the returned records generically.

---

## Further Reading

- [tkinter documentation](https://docs.python.org/3/library/tkinter.html)
- [ttk widgets](https://docs.python.org/3/library/tkinter.ttk.html)
- [requests – Quickstart](https://requests.readthedocs.io/en/latest/user/quickstart/)
- [uv documentation](https://docs.astral.sh/uv/)
- [PyInstaller documentation](https://pyinstaller.org/en/stable/)
- [fpm – Effing Package Management](https://fpm.readthedocs.io/)
- [Inno Setup documentation](https://jrsoftware.org/ishelp/)
- [create-dmg](https://github.com/create-dmg/create-dmg)
- Factory Demo API – Swagger UI: <http://128.140.85.215:8888/docs>
- Lecture 10 handout
