import os
import sys
import time
import random
import string
import ctypes
import logging
import psutil
import pymem
import pymem.process
from ctypes import wintypes
from datetime import datetime
import tkinter as tk
from tkinter import messagebox, simpledialog

# Set up logging
logging.basicConfig(
    filename='sv_cheats_bypass_log.txt',
    level=logging.DEBUG,  # Changed to DEBUG for more detailed logs
    format='%(asctime)s - %(levelname)s - %(message)s',
    filemode='w'
)

# Constants for Windows API
WH_KEYBOARD_LL = 13
WM_KEYDOWN = 0x0100
VK_INSERT = 0x2D

# Global variables
bypass_active = False
keyboard_hook = None
keyboard_hook_pointer = None  # Keep reference to prevent garbage collection
pm = None  # Pymem instance
game_process = None
sv_cheats_address = None
original_flags = 0
original_value = 0

# ConVar offsets - these may need adjustment for different games/versions
CONVAR_NAME_OFFSET = 0x0C
CONVAR_FLAGS_OFFSET = 0x14
CONVAR_VALUE_OFFSET = 0x30
CONVAR_FLOAT_VALUE_OFFSET = 0x2C
CONVAR_STRING_OFFSET = 0x24
CONVAR_NEXT_OFFSET = 0x4  # Offset to the next ConVar in the linked list

# Flag definitions
FCVAR_UNREGISTERED = (1 << 0)
FCVAR_DEVELOPMENTONLY = (1 << 1)
FCVAR_GAMEDLL = (1 << 2)
FCVAR_CLIENTDLL = (1 << 3)
FCVAR_CHEAT = (1 << 4)
FCVAR_PROTECTED = (1 << 5)
FCVAR_SPONLY = (1 << 6)
FCVAR_ARCHIVE = (1 << 7)
FCVAR_REPLICATED = (1 << 8)

def random_string(length):
    """Generate a random string of fixed length"""
    letters = string.ascii_letters
    return ''.join(random.choice(letters) for _ in range(length))

def find_game_process(game_names):
    """Find the game process by name"""
    try:
        for proc in psutil.process_iter(['pid', 'name']):
            for game_name in game_names:
                if game_name.lower() in proc.info['name'].lower():
                    logging.info(f"Found game process: {proc.info['name']} (PID: {proc.info['pid']})")
                    return proc.info['pid']
        return None
    except Exception as e:
        logging.error(f"Error finding game process: {str(e)}")
        return None

def find_pattern(pattern, module_name=None):
    """Find a pattern in memory"""
    try:
        if module_name:
            module = None
            for mod in pm.list_modules():
                if mod.name.lower() == module_name.lower():
                    module = mod
                    break
            
            if not module:
                logging.error(f"Module {module_name} not found")
                return None
            
            logging.info(f"Scanning for pattern in {module_name} at 0x{module.lpBaseOfDll:X}")
            result = pymem.pattern.pattern_scan_module(pm.process_handle, module, pattern)
        else:
            logging.info("Scanning for pattern in entire process memory")
            result = pymem.pattern.pattern_scan_all(pm.process_handle, pattern, return_multiple=True)
            
        return result
    except Exception as e:
        logging.error(f"Error in pattern scanning: {str(e)}")
        return None

def find_cvar_interface():
    """Find the ICvar interface in the game memory"""
    try:
        logging.info("Looking for ICvar interface...")
        
        # Try to find vstdlib.dll module
        vstdlib = None
        for module in pm.list_modules():
            if module.name.lower() == "vstdlib.dll":
                vstdlib = module
                break
        
        if not vstdlib:
            logging.error("vstdlib.dll not found")
            return None
            
        logging.info(f"Found vstdlib.dll at 0x{vstdlib.lpBaseOfDll:X}")
        
        # For demonstration, we'll assume we found the ICvar interface
        logging.info("Successfully found ICvar interface (simulated)")
        return True
    except Exception as e:
        logging.error(f"Error finding ICvar interface: {str(e)}")
        return None

def find_convar_by_traversal(start_convar, name):
    """Find a ConVar by traversing the ConVar linked list"""
    try:
        current = start_convar
        count = 0
        max_count = 10000  # Safety limit
        
        while current and count < max_count:
            try:
                # Read the name
                name_ptr = current + CONVAR_NAME_OFFSET
                convar_name = pm.read_string(name_ptr, 32)
                
                logging.debug(f"Checking ConVar at 0x{current:X}: {convar_name}")
                
                if convar_name.lower() == name.lower():
                    logging.info(f"Found ConVar {name} at 0x{current:X}")
                    return current
                
                # Move to next ConVar
                current = pm.read_int(current + CONVAR_NEXT_OFFSET)
                count += 1
            except Exception as e:
                logging.debug(f"Error reading ConVar at 0x{current:X}: {str(e)}")
                break
        
        logging.error(f"ConVar {name} not found after checking {count} ConVars")
        return None
    except Exception as e:
        logging.error(f"Error in ConVar traversal: {str(e)}")
        return None

def find_convar_head():
    """Find the head of the ConVar linked list"""
    try:
        # Try multiple methods to find the ConVar list head
        
        # Method 1: Look for a known ConVar pattern
        patterns = [
            b"sv_cheats\x00",
            b"host_timescale\x00",
            b"developer\x00",
            b"fps_max\x00"
        ]
        
        for pattern in patterns:
            matches = find_pattern(pattern)
            if matches and len(matches) > 0:
                for match in matches:
                    try:
                        # Check if this could be a valid ConVar
                        possible_convar = match - CONVAR_NAME_OFFSET
                        flags = pm.read_int(possible_convar + CONVAR_FLAGS_OFFSET)
                        
                        # If it has reasonable flags, try to find the list head
                        if 0 <= flags <= 0xFFFF:
                            logging.info(f"Found potential ConVar at 0x{possible_convar:X}")
                            
                            # Try to find the head by traversing backwards
                            # This is a simplified approach and may not work in all games
                            return possible_convar
                    except Exception as e:
                        logging.debug(f"Error checking potential ConVar: {str(e)}")
        
        logging.error("Could not find ConVar list head")
        return None
    except Exception as e:
        logging.error(f"Error finding ConVar head: {str(e)}")
        return None

def find_convar(name):
    """Find a ConVar by name in the game memory"""
    try:
        logging.info(f"Looking for ConVar: {name}")
        
        # Method 1: Direct pattern scan for the ConVar name
        pattern = name.encode('ascii') + b'\x00'
        matches = find_pattern(pattern)
        
        if matches and len(matches) > 0:
            logging.info(f"Found {len(matches)} potential matches for {name}")
            
            for match in matches:
                try:
                    # Check if this is a valid ConVar by reading its structure
                    possible_convar_addr = match - CONVAR_NAME_OFFSET
                    flags = pm.read_int(possible_convar_addr + CONVAR_FLAGS_OFFSET)
                    
                    # If it has reasonable flags, it might be our ConVar
                    if 0 <= flags <= 0xFFFF:
                        value = pm.read_int(possible_convar_addr + CONVAR_VALUE_OFFSET)
                        logging.info(f"Found potential {name} at 0x{possible_convar_addr:X}, value={value}, flags=0x{flags:X}")
                        
                        # Additional validation: try to read the name back
                        try:
                            read_name = pm.read_string(possible_convar_addr + CONVAR_NAME_OFFSET, 32)
                            if read_name.lower() == name.lower():
                                logging.info(f"Validated {name} ConVar at 0x{possible_convar_addr:X}")
                                return possible_convar_addr
                        except:
                            pass
                except Exception as e:
                    logging.debug(f"Error checking potential ConVar at 0x{match:X}: {str(e)}")
        
        # Method 2: Try to find the ConVar list head and traverse it
        logging.info("Trying to find ConVar through list traversal")
        head = find_convar_head()
        if head:
            result = find_convar_by_traversal(head, name)
            if result:
                return result
        
        # Method 3: Try alternative patterns for different game versions
        logging.info("Trying alternative patterns")
        alt_patterns = [
            bytes([0x00]) + name.encode('ascii') + bytes([0x00]),  # Null byte before name
            name.encode('ascii') + bytes([0x00, 0x00, 0x00, 0x00])  # Multiple null bytes after
        ]
        
        for pattern in alt_patterns:
            matches = find_pattern(pattern)
            if matches and len(matches) > 0:
                for match in matches:
                    try:
                        # Adjust the offset based on the pattern
                        offset = 1 if pattern[0] == 0 else 0
                        possible_convar_addr = (match + offset) - CONVAR_NAME_OFFSET
                        flags = pm.read_int(possible_convar_addr + CONVAR_FLAGS_OFFSET)
                        
                        if 0 <= flags <= 0xFFFF:
                            value = pm.read_int(possible_convar_addr + CONVAR_VALUE_OFFSET)
                            logging.info(f"Found potential {name} with alt pattern at 0x{possible_convar_addr:X}")
                            return possible_convar_addr
                    except:
                        pass
        
        logging.error(f"Could not find valid {name} ConVar")
        return None
    except Exception as e:
        logging.error(f"Error finding ConVar {name}: {str(e)}")
        return None

def enable_sv_cheats_bypass():
    global bypass_active, original_flags, original_value
    
    try:
        if not sv_cheats_address:
            messagebox.showerror("Error", "sv_cheats ConVar not found!")
            return
            
        logging.info("Attempting to enable sv_cheats bypass...")
        
        # Save original values
        original_flags = pm.read_int(sv_cheats_address + CONVAR_FLAGS_OFFSET)
        original_value = pm.read_int(sv_cheats_address + CONVAR_VALUE_OFFSET)
        
        logging.info(f"Original sv_cheats flags: 0x{original_flags:X}, value: {original_value}")
        
        # Remove protection flags
        new_flags = original_flags
        new_flags &= ~FCVAR_CHEAT
        new_flags &= ~FCVAR_REPLICATED
        new_flags &= ~FCVAR_PROTECTED
        new_flags &= ~FCVAR_SPONLY
        
        # Write the new flags to memory
        pm.write_int(sv_cheats_address + CONVAR_FLAGS_OFFSET, new_flags)
        
        # Set sv_cheats to 1
        pm.write_int(sv_cheats_address + CONVAR_VALUE_OFFSET, 1)
        pm.write_float(sv_cheats_address + CONVAR_FLOAT_VALUE_OFFSET, 1.0)
        
        # Verify the changes
        current_flags = pm.read_int(sv_cheats_address + CONVAR_FLAGS_OFFSET)
        current_value = pm.read_int(sv_cheats_address + CONVAR_VALUE_OFFSET)
        
        logging.info(f"After bypass: sv_cheats flags = 0x{current_flags:X}, value = {current_value}")
        
        bypass_active = True
        messagebox.showinfo("Success", "sv_cheats bypass ENABLED")
    except Exception as e:
        logging.error(f"Exception in enable_sv_cheats_bypass: {str(e)}")
        messagebox.showerror("Error", f"Failed to enable bypass: {str(e)}")

def disable_sv_cheats_bypass():
    global bypass_active
    
    try:
        if not sv_cheats_address:
            return
            
        logging.info("Attempting to disable sv_cheats bypass...")
        
        # Restore original flags and value
        pm.write_int(sv_cheats_address + CONVAR_FLAGS_OFFSET, original_flags)
        pm.write_int(sv_cheats_address + CONVAR_VALUE_OFFSET, original_value)
        pm.write_float(sv_cheats_address + CONVAR_FLOAT_VALUE_OFFSET, float(original_value))
        
        # Verify the changes
        current_flags = pm.read_int(sv_cheats_address + CONVAR_FLAGS_OFFSET)
        current_value = pm.read_int(sv_cheats_address + CONVAR_VALUE_OFFSET)
        
        logging.info(f"After restore: sv_cheats flags = 0x{current_flags:X}, value = {current_value}")
        
        bypass_active = False
        messagebox.showinfo("Status", "sv_cheats bypass DISABLED")
    except Exception as e:
        logging.error(f"Exception in disable_sv_cheats_bypass: {str(e)}")

def toggle_sv_cheats_bypass():
    try:
        logging.info("Toggle function called")
        if not bypass_active:
            enable_sv_cheats_bypass()
        else:
            disable_sv_cheats_bypass()
    except Exception as e:
        logging.error(f"Exception in toggle_sv_cheats_bypass: {str(e)}")

# Keyboard hook callback function
def low_level_keyboard_handler(nCode, wParam, lParam):
    try:
        if nCode >= 0 and wParam == WM_KEYDOWN:
            kb_struct = ctypes.cast(lParam, ctypes.POINTER(ctypes.c_ulong))
            vk_code = kb_struct[0]
         
            # Check for INSERT key
            if vk_code == VK_INSERT:
                logging.info("INSERT key detected!")
                # Use after to avoid blocking the message loop
                if tk._default_root:
                    tk._default_root.after(1, toggle_sv_cheats_bypass)
    except Exception as e:
        logging.error(f"Exception in keyboard handler: {str(e)}")
    
    # Call the next hook
    return ctypes.windll.user32.CallNextHookEx(keyboard_hook, nCode, wParam, lParam)

def set_keyboard_hook():
    global keyboard_hook, keyboard_hook_pointer
    
    try:
        logging.info("Setting up keyboard hook")
        
        # Define the callback function type
        CMPFUNC = ctypes.CFUNCTYPE(ctypes.c_int, ctypes.c_int, ctypes.c_int, ctypes.POINTER(ctypes.c_void_p))
        keyboard_hook_pointer = CMPFUNC(low_level_keyboard_handler)
        
        # Set the hook
        keyboard_hook = ctypes.windll.user32.SetWindowsHookExW(
            WH_KEYBOARD_LL, 
            keyboard_hook_pointer, 
            ctypes.windll.kernel32.GetModuleHandleW(None), 
            0
        )
        
        if not keyboard_hook:
            error_code = ctypes.windll.kernel32.GetLastError()
            logging.error(f"Failed to set keyboard hook! Error code: {error_code}")
            messagebox.showerror("Error", f"Failed to set keyboard hook! Error code: {error_code}")
            return False
            
        logging.info("Keyboard hook set up successfully")
        return True
    except Exception as e:
        logging.error(f"Exception in set_keyboard_hook: {str(e)}")
        return False

def cleanup():
    global keyboard_hook, pm
    
    try:
        logging.info("Cleaning up resources")
        
        # Disable the bypass if active
        if bypass_active:
            disable_sv_cheats_bypass()
        
        # Unhook keyboard hook
        if keyboard_hook:
            ctypes.windll.user32.UnhookWindowsHookEx(keyboard_hook)
            keyboard_hook = None
        
        # Close Pymem instance
        if pm:
            pm.close_process()
            
        logging.info("Cleanup complete")
    except Exception as e:
        logging.error(f"Exception in cleanup: {str(e)}")

def initialize():
    global pm, sv_cheats_address
    
    try:
        # Clear log file
        with open('sv_cheats_bypass_log.txt', 'w') as f:
            f.write("Initializing sv_cheats bypass plugin...\n")
        
        # Set random seed
        random.seed(time.time())
        
        # Find game process
        game_names = [
            "hl2.exe", "csgo.exe", "portal2.exe", "left4dead2.exe", 
            "tf_win64.exe", "tf2.exe", "hl2_win64.exe", "csgo_win64.exe",
            "portal2_win64.exe", "left4dead2_win64.exe", "dota2.exe"
        ]
        game_pid = find_game_process(game_names)
        
        if not game_pid:
            logging.error("Game process not found!")
            messagebox.showerror("Error", "Game process not found! Make sure the game is running.")
            return False
        
        # Open the process
        try:
            pm = pymem.Pymem(game_pid)
            logging.info(f"Successfully opened game process (PID: {game_pid})")
        except Exception as e:
            logging.error(f"Failed to open game process: {str(e)}")
            messagebox.showerror("Error", f"Failed to open game process: {str(e)}")
            return False
        
        # Find ICvar interface
        if not find_cvar_interface():
            logging.warning("Failed to find ICvar interface! Continuing anyway...")
        
        # Find sv_cheats ConVar
        sv_cheats_address = find_convar("sv_cheats")
        if not sv_cheats_address:
            logging.error("Failed to find sv_cheats ConVar!")
            messagebox.showerror("Error", "Failed to find sv_cheats ConVar! Make sure you're running a Source Engine game.")
            return False
        
        # Set up keyboard hook
        if not set_keyboard_hook():
            return False
        
        messagebox.showinfo("Plugin Loaded", "sv_cheats Bypass Plugin Loaded!\nPress INSERT key to toggle bypass.")
        return True
    except Exception as e:
        logging.error(f"Exception in initialize: {str(e)}")
        messagebox.showerror("Error", f"Initialization error: {str(e)}")
        return False

class SvCheatsApp:
    def __init__(self, root):
        self.root = root
        self.root.title("sv_cheats Bypass Tool")
        self.root.geometry("400x300")
        self.root.resizable(False, False)
        
        # Create UI elements
        self.label = tk.Label(root, text="sv_cheats Bypass Tool", font=("Arial", 14, "bold"))
        self.label.pack(pady=10)
        
        self.game_label = tk.Label(root, text="Game: Not connected", font=("Arial", 10))
        self.game_label.pack(pady=5)
        
        self.status_label = tk.Label(root, text="Status: Inactive", font=("Arial", 12))
        self.status_label.pack(pady=5)
        
        self.value_label = tk.Label(root, text="sv_cheats value: Unknown", font=("Arial", 10))
        self.value_label.pack(pady=5)
        
        self.flags_label = tk.Label(root, text="Flags: Unknown", font=("Arial", 10))
        self.flags_label.pack(pady=5)
        
        # Buttons frame
        self.button_frame = tk.Frame(root)
        self.button_frame.pack(pady=10)
        
        self.toggle_button = tk.Button(self.button_frame, text="Toggle Bypass", command=self.toggle_bypass, width=15)
        self.toggle_button.grid(row=0, column=0, padx=5)
        
        self.refresh_button = tk.Button(self.button_frame, text="Refresh Status", command=self.refresh_status, width=15)
        self.refresh_button.grid(row=0, column=1, padx=5)
        
        # Advanced options frame
        self.advanced_frame = tk.LabelFrame(root, text="Advanced Options")
        self.advanced_frame.pack(pady=10, padx=10, fill="x")
        
        self.set_value_button = tk.Button(self.advanced_frame, text="Set Custom Value", command=self.set_custom_value)
        self.set_value_button.pack(side=tk.LEFT, padx=5, pady=5)
        
        self.dump_info_button = tk.Button(self.advanced_frame, text="Dump ConVar Info", command=self.dump_convar_info)
        self.dump_info_button.pack(side=tk.RIGHT, padx=5, pady=5)
        
        # Info label
        self.info_label = tk.Label(root, text="Press INSERT key to toggle bypass", font=("Arial", 10, "italic"))
        self.info_label.pack(pady=10)
        
        # Set up cleanup on window close
        self.root.protocol("WM_DELETE_WINDOW", self.on_close)
        
        # Update status periodically
        self.update_status()
    
    def toggle_bypass(self):
        toggle_sv_cheats_bypass()
        self.update_status()
    
    def refresh_status(self):
        self.update_status(force=True)
    
    def set_custom_value(self):
        if not sv_cheats_address:
            messagebox.showerror("Error", "sv_cheats ConVar not found!")
            return
            
        try:
            value = simpledialog.askinteger("Set sv_cheats Value", "Enter value for sv_cheats:", 
                                           minvalue=0, maxvalue=1000)
            if value is not None:
                logging.info(f"Setting custom sv_cheats value: {value}")
                pm.write_int(sv_cheats_address + CONVAR_VALUE_OFFSET, value)
                pm.write_float(sv_cheats_address + CONVAR_FLOAT_VALUE_OFFSET, float(value))
                self.update_status(force=True)
        except Exception as e:
            logging.error(f"Error setting custom value: {str(e)}")
            messagebox.showerror("Error", f"Failed to set value: {str(e)}")
    
    def dump_convar_info(self):
        if not sv_cheats_address:
            messagebox.showerror("Error", "sv_cheats ConVar not found!")
            return
            
        try:
            # Read ConVar structure details
            flags = pm.read_int(sv_cheats_address + CONVAR_FLAGS_OFFSET)
            value = pm.read_int(sv_cheats_address + CONVAR_VALUE_OFFSET)
            float_value = pm.read_float(sv_cheats_address + CONVAR_FLOAT_VALUE_OFFSET)
            
            # Try to read string value and name
            try:
                string_ptr = pm.read_int(sv_cheats_address + CONVAR_STRING_OFFSET)
                string_value = pm.read_string(string_ptr, 32)
            except:
                string_value = "Unable to read"
                
            try:
                name_ptr = sv_cheats_address + CONVAR_NAME_OFFSET
                name = pm.read_string(name_ptr, 32)
            except:
                name = "sv_cheats"
            
            # Format the flags as binary for better understanding
            flags_binary = format(flags, '032b')
            flags_formatted = ' '.join(flags_binary[i:i+4] for i in range(0, len(flags_binary), 4))
            
            info = (
                f"ConVar: {name}\n"
                f"Address: 0x{sv_cheats_address:X}\n"
                f"Int Value: {value}\n"
                f"Float Value: {float_value}\n"
                f"String Value: {string_value}\n"
                f"Flags (Hex): 0x{flags:X}\n"
                f"Flags (Binary): {flags_formatted}\n\n"
                f"Flag Meanings:\n"
                f"FCVAR_UNREGISTERED (1): {bool(flags & FCVAR_UNREGISTERED)}\n"
                f"FCVAR_DEVELOPMENTONLY (2): {bool(flags & FCVAR_DEVELOPMENTONLY)}\n"
                f"FCVAR_GAMEDLL (4): {bool(flags & FCVAR_GAMEDLL)}\n"
                f"FCVAR_CLIENTDLL (8): {bool(flags & FCVAR_CLIENTDLL)}\n"
                f"FCVAR_CHEAT (16): {bool(flags & FCVAR_CHEAT)}\n"
                f"FCVAR_PROTECTED (32): {bool(flags & FCVAR_PROTECTED)}\n"
                f"FCVAR_SPONLY (64): {bool(flags & FCVAR_SPONLY)}\n"
                f"FCVAR_ARCHIVE (128): {bool(flags & FCVAR_ARCHIVE)}\n"
                f"FCVAR_REPLICATED (256): {bool(flags & FCVAR_REPLICATED)}"
            )
            
            # Log the info
            logging.info(f"ConVar Dump:\n{info}")
            
            # Show in a message box
            messagebox.showinfo("ConVar Information", info)
        except Exception as e:
            logging.error(f"Error dumping ConVar info: {str(e)}")
            messagebox.showerror("Error", f"Failed to dump ConVar info: {str(e)}")
    
    def update_status(self, force=False):
        try:
            if sv_cheats_address:
                # Update game process info
                if pm and pm.process_handle:
                    process = psutil.Process(pm.process_id)
                    self.game_label.config(text=f"Game: {process.name()} (PID: {pm.process_id})")
                
                # Update sv_cheats value and flags
                value = pm.read_int(sv_cheats_address + CONVAR_VALUE_OFFSET)
                flags = pm.read_int(sv_cheats_address + CONVAR_FLAGS_OFFSET)
                
                self.value_label.config(text=f"sv_cheats value: {value}")
                self.flags_label.config(text=f"Flags: 0x{flags:X}")
                
                if bypass_active:
                    self.status_label.config(text="Status: BYPASS ACTIVE", fg="green")
                else:
                    self.status_label.config(text="Status: Inactive", fg="black")
            else:
                self.status_label.config(text="Status: ConVar Not Found", fg="red")
        except Exception as e:
            if force:
                logging.error(f"Error updating status: {str(e)}")
                self.status_label.config(text="Status: Error", fg="red")
        
        # Schedule the next update
        self.root.after(1000, self.update_status)
    
    def on_close(self):
        cleanup()
        self.root.destroy()

def process_messages():
    """Process Windows messages to keep the keyboard hook working"""
    msg = wintypes.MSG()
    while ctypes.windll.user32.PeekMessageW(ctypes.byref(msg), None, 0, 0, 1):
        ctypes.windll.user32.TranslateMessage(ctypes.byref(msg))
        ctypes.windll.user32.DispatchMessageW(ctypes.byref(msg))
    
    # Schedule the next message processing
    if tk._default_root:
        tk._default_root.after(10, process_messages)

def main():
    try:
        # Initialize the application
        if not initialize():
            return
        
        # Create the main window
        root = tk.Tk()
        app = SvCheatsApp(root)
        
        # Start message processing for keyboard hook
        process_messages()
        
        # Start the message pump
        root.mainloop()
    except Exception as e:
        logging.error(f"Exception in main: {str(e)}")
        messagebox.showerror("Error", f"An error occurred: {str(e)}")
    finally:
        # Make sure we clean up
        cleanup()

if __name__ == "__main__":
    main()

For batch.


)))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))

@echo off
setlocal enabledelayedexpansion

echo SV_CHEATS Bypass Tool Builder
echo ============================
echo.

REM Check if Python is installed
python --version >nul 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo ERROR: Python is not installed or not in PATH.
    echo Please install Python from https://www.python.org/downloads/
    echo Make sure to check "Add Python to PATH" during installation.
    pause
    exit /b 1
)

REM Check if required packages are installed
echo Checking required packages...
python -c "import pymem" >nul 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Installing pymem...
    pip install pymem
)

python -c "import psutil" >nul 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Installing psutil...
    pip install psutil
)

REM Check if PyInstaller is installed
python -c "import PyInstaller" >nul 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Installing PyInstaller...
    pip install pyinstaller
)

echo All required packages are installed.
echo.

:menu
cls
echo SV_CHEATS Bypass Tool
echo ============================
echo 1. Build executable
echo 2. Run Python script directly
echo 3. Run executable (if already built)
echo 4. Exit
echo.
set /p choice="Enter your choice (1-4): "

if "%choice%"=="1" goto build
if "%choice%"=="2" goto run_script
if "%choice%"=="3" goto run_exe
if "%choice%"=="4" goto end
goto menu

:build
echo.
echo Building executable...
echo.

REM Check if the script exists
if not exist fixed_game_convar_hook.py (
    echo ERROR: fixed_game_convar_hook.py not found in the current directory.
    pause
    goto menu
)

REM Build with PyInstaller
pyinstaller --onefile --noconsole --uac-admin --name "SvCheatsUnlocker" fixed_game_convar_hook.py

if %ERRORLEVEL% NEQ 0 (
    echo.
    echo ERROR: Build failed!
    pause
    goto menu
)

echo.
echo Build successful! Executable created in dist\SvCheatsUnlocker.exe
echo.
pause
goto menu

:run_script
echo.
echo Running Python script directly...
echo.

REM Check if the script exists
if not exist fixed_game_convar_hook.py (
    echo ERROR: fixed_game_convar_hook.py not found in the current directory.
    pause
    goto menu
)

REM Check if a Source engine game is running
tasklist /FI "IMAGENAME eq hl2.exe" /FI "IMAGENAME eq csgo.exe" /FI "IMAGENAME eq portal2.exe" /FI "IMAGENAME eq left4dead2.exe" /FI "IMAGENAME eq tf2.exe" 2>NUL | find /I /N "exe" >NUL
if %ERRORLEVEL% NEQ 0 (
    echo WARNING: No Source engine game appears to be running.
    echo The tool needs a running game to work properly.
    echo.
    set /p continue="Continue anyway? (y/n): "
    if /i not "!continue!"=="y" goto menu
)

REM Run the script with elevated privileges
echo Running script with admin privileges...
powershell -Command "Start-Process python -ArgumentList 'fixed_game_convar_hook.py' -Verb RunAs"

echo.
echo Script started. Check for any error messages.
echo.
pause
goto menu

:run_exe
echo.
echo Running executable...
echo.

REM Check if the executable exists
if not exist dist\SvCheatsUnlocker.exe (
    echo ERROR: Executable not found. Build it first.
    pause
    goto menu
)

REM Check if a Source engine game is running
tasklist /FI "IMAGENAME eq hl2.exe" /FI "IMAGENAME eq csgo.exe" /FI "IMAGENAME eq portal2.exe" /FI "IMAGENAME eq left4dead2.exe" /FI "IMAGENAME eq tf2.exe" 2>NUL | find /I /N "exe" >NUL
if %ERRORLEVEL% NEQ 0 (
    echo WARNING: No Source engine game appears to be running.
    echo The tool needs a running game to work properly.
    echo.
    set /p continue="Continue anyway? (y/n): "
    if /i not "!continue!"=="y" goto menu
)

REM Run the executable with elevated privileges
echo Running executable with admin privileges...
powershell -Command "Start-Process 'dist\SvCheatsUnlocker.exe' -Verb RunAs"

echo.
echo Executable started. Check for any error messages.
echo.
pause
goto menu

:end
echo.
echo Thank you for using SV_CHEATS Bypass Tool.
echo.
exit /b 0


