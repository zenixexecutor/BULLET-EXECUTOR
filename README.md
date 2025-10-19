# BULLET-EXECUTOR
using System;

using System.Diagnostics;

using System.IO;

using System.Net.Http;

using System.Runtime.InteropServices;

using System.Text;

using System.Threading.Tasks;

using System.Windows.Forms;

using ScintillaNET;

using System.Drawing;



public class LuaExecutorForm : Form

{

private Scintilla luaEditor;

private Button executeButton, loadFromUrlButton, injectButton;

private TextBox urlTextBox, dllPathTextBox;

private Label statusLabel, dllLabel;

private PictureBox logoPictureBox;



[DllImport("kernel32.dll", SetLastError = true)]

static extern IntPtr OpenProcess(uint dwDesiredAccess, bool bInheritHandle, int dwProcessId);



[DllImport("kernel32.dll", SetLastError = true)]

static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);



[DllImport("kernel32.dll", SetLastError = true)]

static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] buffer, uint size, out IntPtr lpNumberOfBytesWritten);



[DllImport("kernel32.dll")]

static extern IntPtr CreateRemoteThread(IntPtr hProcess, IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress,

IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);



[DllImport("kernel32.dll", CharSet = CharSet.Auto)]

static extern IntPtr GetModuleHandle(string lpModuleName);



[DllImport("kernel32.dll", CharSet = CharSet.Ansi, ExactSpelling = true, SetLastError = true)]

static extern IntPtr GetProcAddress(IntPtr hModule, string procName);



const uint PROCESS_ALL_ACCESS = 0x001F0FFF;

const uint MEM_COMMIT = 0x00001000;

const uint MEM_RESERVE = 0x00002000;

const uint PAGE_READWRITE = 4;



public LuaExecutorForm()

{

Text = "BULLET Executor";

Width = 600;

Height = 650; // Adjusted height to fit logo

StartPosition = FormStartPosition.CenterScreen;

this.ShowIcon = false;



luaEditor = new Scintilla()

{

Location = new System.Drawing.Point(10, 10),

Size = new System.Drawing.Size(560, 350),

Lexer = Lexer.Lua,

WrapMode = WrapMode.Word,

};

ConfigureLuaHighlighting(luaEditor);



urlTextBox = new TextBox() { Location = new System.Drawing.Point(10, 370), Width = 400, PlaceholderText = "Enter GitHub Raw URL of Lua script here..." };

loadFromUrlButton = new Button() { Text = "Load Lua from URL", Location = new System.Drawing.Point(420, 368), Width = 150 };

loadFromUrlButton.Click += async (s, e) => await LoadLuaFromUrl();



executeButton = new Button() { Text = "Execute Lua", Location = new System.Drawing.Point(230, 410), Width = 120 };

executeButton.Click += ExecuteLua;



dllLabel = new Label() { Text = "DLL Path:", Location = new System.Drawing.Point(10, 460), AutoSize = true };

dllPathTextBox = new TextBox() { Location = new System.Drawing.Point(70, 457), Width = 400, PlaceholderText = "Enter full path to DLL to inject" };

injectButton = new Button() { Text = "Inject DLL", Location = new System.Drawing.Point(480, 455), Width = 90 };

injectButton.Click += InjectButton_Click;



statusLabel = new Label() { Location = new System.Drawing.Point(10, 500), Width = 560, ForeColor = System.Drawing.Color.DarkGreen, AutoSize = false };



// Add controls to form

Controls.Add(luaEditor);

Controls.Add(urlTextBox);

Controls.Add(loadFromUrlButton);

Controls.Add(executeButton);

Controls.Add(dllLabel);

Controls.Add(dllPathTextBox);

Controls.Add(injectButton);

Controls.Add(statusLabel);



// Add PictureBox for logo image

logoPictureBox = new PictureBox()

{

Location = new System.Drawing.Point(10, 530), // Position at bottom left

Size = new System.Drawing.Size(100, 100), // Size to fit your logo

SizeMode = PictureBoxSizeMode.StretchImage,

BorderStyle = BorderStyle.FixedSingle

};



// Load logo image - ensure image.jpg is in working directory or provide full path

logoPictureBox.Image = Image.FromFile("image.jpg");

Controls.Add(logoPictureBox);



Timer antiDebugTimer = new Timer() { Interval = 3000 };

antiDebugTimer.Tick += (s, e) =>

{

if (Debugger.IsAttached)

{

MessageBox.Show("Debugging detected. The program will exit.");

Environment.Exit(0);

}

};

antiDebugTimer.Start();

}



private void ConfigureLuaHighlighting(Scintilla scintilla)

{

scintilla.StyleResetDefault();

scintilla.Styles[Style.Lua.Default].ForeColor = System.Drawing.Color.Silver;

scintilla.Styles[Style.Lua.Comment].ForeColor = System.Drawing.Color.Green;

scintilla.Styles[Style.Lua.Number].ForeColor = System.Drawing.Color.Olive;

scintilla.Styles[Style.Lua.String].ForeColor = System.Drawing.Color.Maroon;

scintilla.Styles[Style.Lua.Character].ForeColor = System.Drawing.Color.Maroon;

scintilla.Styles[Style.Lua.Word].ForeColor = System.Drawing.Color.Blue;

scintilla.Styles[Style.Lua.Word2].ForeColor = System.Drawing.Color.Blue;

scintilla.Styles[Style.Lua.Operator].ForeColor = System.Drawing.Color.Purple;



scintilla.SetKeywords(0, "and break do else elseif end false for function if in local nil not or repeat return then true until while");

scintilla.SetKeywords(1, "print wait game workspace script player players");

}



private async Task LoadLuaFromUrl()

{

string url = urlTextBox.Text.Trim();

if (!Uri.TryCreate(url, UriKind.Absolute, out _) ||

!(url.StartsWith("https://raw.githubusercontent.com") || url.StartsWith("https://gist.githubusercontent.com")))

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "Please enter a valid raw GitHub or Gist URL.";

return;

}

try

{

using HttpClient client = new HttpClient();

string luaCode = await client.GetStringAsync(url);

luaEditor.Text = luaCode;

statusLabel.ForeColor = System.Drawing.Color.Green;

statusLabel.Text = "Lua script loaded successfully.";

}

catch (Exception ex)

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = $"Failed to load script: {ex.Message}";

}

}



private void ExecuteLua(object sender, EventArgs e)

{

string luaCode = luaEditor.Text.Trim();

if (string.IsNullOrEmpty(luaCode))

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "No Lua code to execute.";

return;

}

statusLabel.ForeColor = System.Drawing.Color.Green;

statusLabel.Text = "Lua script executed (simulation).";

}



private void InjectButton_Click(object sender, EventArgs e)

{

string dllPath = dllPathTextBox.Text.Trim();

if (string.IsNullOrEmpty(dllPath) || !File.Exists(dllPath))

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "DLL path is empty or file does not exist.";

return;

}

Process[] processes = Process.GetProcessesByName("RobloxPlayerBeta");

if (processes.Length == 0)

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "Roblox process not found.";

return;

}

Process process = processes[0];

try

{

if (InjectDll(process, dllPath))

{

statusLabel.ForeColor = System.Drawing.Color.Green;

statusLabel.Text = "DLL injected successfully!";

}

else

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "DLL injection failed.";

}

}

catch (Exception ex)

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = $"Injection error: {ex.Message}";

}

}



private bool InjectDll(Process targetProcess, string dllPath)

{

IntPtr procHandle = OpenProcess(PROCESS_ALL_ACCESS, false, targetProcess.Id);

if (procHandle == IntPtr.Zero)

throw new Exception("Failed to open target process.");



IntPtr allocMemAddress = VirtualAllocEx(procHandle, IntPtr.Zero, (uint)((dllPath.Length + 1) * Marshal.SizeOf(typeof(char))),

MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);

if (allocMemAddress == IntPtr.Zero)

throw new Exception("Failed to allocate memory in target process.");



byte[] bytes = Encoding.Unicode.GetBytes(dllPath);

if (!WriteProcessMemory(procHandle, allocMemAddress, bytes, (uint)bytes.Length, out IntPtr bytesWritten) || bytesWritten == IntPtr.Zero)

throw new Exception("Failed to write DLL path to target process.");



IntPtr kernel32Handle = GetModuleHandle("kernel32.dll");

IntPtr loadLibraryAddr = GetProcAddress(kernel32Handle, "LoadLibraryW");

if (loadLibraryAddr == IntPtr.Zero)

throw new Exception("Failed to get LoadLibrary address.");



IntPtr threadHandle = CreateRemoteThread(procHandle, IntPtr.Zero, 0, loadLibraryAddr, allocMemAddress, 0, IntPtr.Zero);

if (threadHandle == IntPtr.Zero)

throw new Exception("Failed to create remote thread.");



return true;

}



[STAThread]

static void Main()

{

Application.EnableVisualStyles();

Application.SetCompatibleTextRenderingDefault(false);

Application.Run(new LuaExecutorForm());

}

}


using System;

using System.Diagnostics;

using System.IO;

using System.Net.Http;

using System.Runtime.InteropServices;

using System.Text;

using System.Threading.Tasks;

using System.Windows.Forms;

using ScintillaNET;

using System.Drawing;



public class LuaExecutorForm : Form

{

private Scintilla luaEditor;

private Button executeButton, loadFromUrlButton, injectButton;

private TextBox urlTextBox, dllPathTextBox;

private Label statusLabel, dllLabel;

private PictureBox logoPictureBox;



[DllImport("kernel32.dll", SetLastError = true)]

static extern IntPtr OpenProcess(uint dwDesiredAccess, bool bInheritHandle, int dwProcessId);



[DllImport("kernel32.dll", SetLastError = true)]

static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);



[DllImport("kernel32.dll", SetLastError = true)]

static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] buffer, uint size, out IntPtr lpNumberOfBytesWritten);



[DllImport("kernel32.dll")]

static extern IntPtr CreateRemoteThread(IntPtr hProcess, IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress,

IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);



[DllImport("kernel32.dll", CharSet = CharSet.Auto)]

static extern IntPtr GetModuleHandle(string lpModuleName);



[DllImport("kernel32.dll", CharSet = CharSet.Ansi, ExactSpelling = true, SetLastError = true)]

static extern IntPtr GetProcAddress(IntPtr hModule, string procName);



const uint PROCESS_ALL_ACCESS = 0x001F0FFF;

const uint MEM_COMMIT = 0x00001000;

const uint MEM_RESERVE = 0x00002000;

const uint PAGE_READWRITE = 4;



public LuaExecutorForm()

{

Text = "BULLET Executor";

Width = 600;

Height = 650; // Adjusted height to fit logo

StartPosition = FormStartPosition.CenterScreen;

this.ShowIcon = false;



luaEditor = new Scintilla()

{

Location = new System.Drawing.Point(10, 10),

Size = new System.Drawing.Size(560, 350),

Lexer = Lexer.Lua,

WrapMode = WrapMode.Word,

};

ConfigureLuaHighlighting(luaEditor);



urlTextBox = new TextBox() { Location = new System.Drawing.Point(10, 370), Width = 400, PlaceholderText = "Enter GitHub Raw URL of Lua script here..." };

loadFromUrlButton = new Button() { Text = "Load Lua from URL", Location = new System.Drawing.Point(420, 368), Width = 150 };

loadFromUrlButton.Click += async (s, e) => await LoadLuaFromUrl();



executeButton = new Button() { Text = "Execute Lua", Location = new System.Drawing.Point(230, 410), Width = 120 };

executeButton.Click += ExecuteLua;



dllLabel = new Label() { Text = "DLL Path:", Location = new System.Drawing.Point(10, 460), AutoSize = true };

dllPathTextBox = new TextBox() { Location = new System.Drawing.Point(70, 457), Width = 400, PlaceholderText = "Enter full path to DLL to inject" };

injectButton = new Button() { Text = "Inject DLL", Location = new System.Drawing.Point(480, 455), Width = 90 };

injectButton.Click += InjectButton_Click;



statusLabel = new Label() { Location = new System.Drawing.Point(10, 500), Width = 560, ForeColor = System.Drawing.Color.DarkGreen, AutoSize = false };



// Add controls to form

Controls.Add(luaEditor);

Controls.Add(urlTextBox);

Controls.Add(loadFromUrlButton);

Controls.Add(executeButton);

Controls.Add(dllLabel);

Controls.Add(dllPathTextBox);

Controls.Add(injectButton);

Controls.Add(statusLabel);



// Add PictureBox for logo image

logoPictureBox = new PictureBox()

{

Location = new System.Drawing.Point(10, 530), // Position at bottom left

Size = new System.Drawing.Size(100, 100), // Size to fit your logo

SizeMode = PictureBoxSizeMode.StretchImage,

BorderStyle = BorderStyle.FixedSingle

};



// Load logo image - ensure image.jpg is in working directory or provide full path

logoPictureBox.Image = Image.FromFile("image.jpg");

Controls.Add(logoPictureBox);



Timer antiDebugTimer = new Timer() { Interval = 3000 };

antiDebugTimer.Tick += (s, e) =>

{

if (Debugger.IsAttached)

{

MessageBox.Show("Debugging detected. The program will exit.");

Environment.Exit(0);

}

};

antiDebugTimer.Start();

}



private void ConfigureLuaHighlighting(Scintilla scintilla)

{

scintilla.StyleResetDefault();

scintilla.Styles[Style.Lua.Default].ForeColor = System.Drawing.Color.Silver;

scintilla.Styles[Style.Lua.Comment].ForeColor = System.Drawing.Color.Green;

scintilla.Styles[Style.Lua.Number].ForeColor = System.Drawing.Color.Olive;

scintilla.Styles[Style.Lua.String].ForeColor = System.Drawing.Color.Maroon;

scintilla.Styles[Style.Lua.Character].ForeColor = System.Drawing.Color.Maroon;

scintilla.Styles[Style.Lua.Word].ForeColor = System.Drawing.Color.Blue;

scintilla.Styles[Style.Lua.Word2].ForeColor = System.Drawing.Color.Blue;

scintilla.Styles[Style.Lua.Operator].ForeColor = System.Drawing.Color.Purple;



scintilla.SetKeywords(0, "and break do else elseif end false for function if in local nil not or repeat return then true until while");

scintilla.SetKeywords(1, "print wait game workspace script player players");

}



private async Task LoadLuaFromUrl()

{

string url = urlTextBox.Text.Trim();

if (!Uri.TryCreate(url, UriKind.Absolute, out _) ||

!(url.StartsWith("https://raw.githubusercontent.com") || url.StartsWith("https://gist.githubusercontent.com")))

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "Please enter a valid raw GitHub or Gist URL.";

return;

}

try

{

using HttpClient client = new HttpClient();

string luaCode = await client.GetStringAsync(url);

luaEditor.Text = luaCode;

statusLabel.ForeColor = System.Drawing.Color.Green;

statusLabel.Text = "Lua script loaded successfully.";

}

catch (Exception ex)

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = $"Failed to load script: {ex.Message}";

}

}



private void ExecuteLua(object sender, EventArgs e)

{

string luaCode = luaEditor.Text.Trim();

if (string.IsNullOrEmpty(luaCode))

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "No Lua code to execute.";

return;

}

statusLabel.ForeColor = System.Drawing.Color.Green;

statusLabel.Text = "Lua script executed (simulation).";

}



private void InjectButton_Click(object sender, EventArgs e)

{

string dllPath = dllPathTextBox.Text.Trim();

if (string.IsNullOrEmpty(dllPath) || !File.Exists(dllPath))

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "DLL path is empty or file does not exist.";

return;

}

Process[] processes = Process.GetProcessesByName("RobloxPlayerBeta");

if (processes.Length == 0)

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "Roblox process not found.";

return;

}

Process process = processes[0];

try

{

if (InjectDll(process, dllPath))

{

statusLabel.ForeColor = System.Drawing.Color.Green;

statusLabel.Text = "DLL injected successfully!";

}

else

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = "DLL injection failed.";

}

}

catch (Exception ex)

{

statusLabel.ForeColor = System.Drawing.Color.Red;

statusLabel.Text = $"Injection error: {ex.Message}";

}

}



private bool InjectDll(Process targetProcess, string dllPath)

{

IntPtr procHandle = OpenProcess(PROCESS_ALL_ACCESS, false, targetProcess.Id);

if (procHandle == IntPtr.Zero)

throw new Exception("Failed to open target process.");



IntPtr allocMemAddress = VirtualAllocEx(procHandle, IntPtr.Zero, (uint)((dllPath.Length + 1) * Marshal.SizeOf(typeof(char))),

MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);

if (allocMemAddress == IntPtr.Zero)

throw new Exception("Failed to allocate memory in target process.");



byte[] bytes = Encoding.Unicode.GetBytes(dllPath);

if (!WriteProcessMemory(procHandle, allocMemAddress, bytes, (uint)bytes.Length, out IntPtr bytesWritten) || bytesWritten == IntPtr.Zero)

throw new Exception("Failed to write DLL path to target process.");



IntPtr kernel32Handle = GetModuleHandle("kernel32.dll");

IntPtr loadLibraryAddr = GetProcAddress(kernel32Handle, "LoadLibraryW");

if (loadLibraryAddr == IntPtr.Zero)

throw new Exception("Failed to get LoadLibrary address.");



IntPtr threadHandle = CreateRemoteThread(procHandle, IntPtr.Zero, 0, loadLibraryAddr, allocMemAddress, 0, IntPtr.Zero);

if (threadHandle == IntPtr.Zero)

throw new Exception("Failed to create remote thread.");



return true;

}



[STAThread]

static void Main()

{

Application.EnableVisualStyles();

Application.SetCompatibleTextRenderingDefault(false);

Application.Run(new LuaExecutorForm());

}


