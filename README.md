import tkinter as tk
from tkinter import ttk, filedialog, messagebox
import os
import subprocess
import sys
from pathlib import Path
import threading
from datetime import datetime
import webbrowser

class FileBrowserGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("Advanced File Browser - PDF/DWG Viewer")
        self.root.geometry("1200x800")
        
        # Store selected folders and files
        self.selected_folders = []
        self.current_files = []
        
        # Configure style
        style = ttk.Style()
        style.theme_use('clam')
        
        self.setup_gui()
        
    def setup_gui(self):
        # Main container
        main_frame = ttk.Frame(self.root)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # Top controls frame
        controls_frame = ttk.Frame(main_frame)
        controls_frame.pack(fill=tk.X, pady=(0, 10))
        
        # Drive/Path selection
        ttk.Label(controls_frame, text="Select Drive/Path:").pack(side=tk.LEFT, padx=(0, 5))
        
        self.path_var = tk.StringVar()
        self.path_entry = ttk.Entry(controls_frame, textvariable=self.path_var, width=60)
        self.path_entry.pack(side=tk.LEFT, padx=(0, 5))
        
        ttk.Button(controls_frame, text="Browse", command=self.browse_path).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(controls_frame, text="Load Folders", command=self.load_folders).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(controls_frame, text="Refresh", command=self.refresh_view).pack(side=tk.LEFT)
        
        # Add a "Select All" and "Clear All" buttons
        select_buttons_frame = ttk.Frame(controls_frame)
        select_buttons_frame.pack(side=tk.RIGHT, padx=(10, 0))
        
        ttk.Button(select_buttons_frame, text="Select All Folders", command=self.select_all_folders).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(select_buttons_frame, text="Clear Selection", command=self.clear_selection).pack(side=tk.LEFT)
        
        # Add scan depth control
        depth_frame = ttk.Frame(controls_frame)
        depth_frame.pack(side=tk.RIGHT, padx=(10, 0))
        
        ttk.Label(depth_frame, text="Scan Depth:").pack(side=tk.LEFT, padx=(0, 5))
        self.depth_var = tk.StringVar(value="Quick (2 levels)")
        depth_combo = ttk.Combobox(depth_frame, textvariable=self.depth_var, 
                                 values=["Quick (2 levels)", "Medium (3 levels)", "Deep (5 levels)"], 
                                 width=15, state="readonly")
        depth_combo.pack(side=tk.LEFT)
        
        # Main content area with paned window
        paned_window = ttk.PanedWindow(main_frame, orient=tk.HORIZONTAL)
        paned_window.pack(fill=tk.BOTH, expand=True)
        
        # Left panel - Folder tree
        left_frame = ttk.Frame(paned_window)
        paned_window.add(left_frame, weight=1)
        
        ttk.Label(left_frame, text="Folders", font=('Arial', 10, 'bold')).pack(anchor=tk.W)
        
        # Folder tree with checkboxes
        self.folder_tree = ttk.Treeview(left_frame, show='tree headings', selectmode='extended')
        self.folder_tree.heading('#0', text='Select Folders', anchor=tk.W)
        
        folder_scrollbar = ttk.Scrollbar(left_frame, orient=tk.VERTICAL, command=self.folder_tree.yview)
        self.folder_tree.configure(yscrollcommand=folder_scrollbar.set)
        
        self.folder_tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        folder_scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Bind events
        self.folder_tree.bind('<Button-1>', self.on_folder_click)
        self.folder_tree.bind('<Double-1>', self.on_folder_double_click)
        
        # Right panel - File list and preview
        right_frame = ttk.Frame(paned_window)
        paned_window.add(right_frame, weight=2)
        
        # File list frame
        file_frame = ttk.Frame(right_frame)
        file_frame.pack(fill=tk.BOTH, expand=True)
        
        # File list label
        file_label_frame = ttk.Frame(file_frame)
        file_label_frame.pack(fill=tk.X)
        ttk.Label(file_label_frame, text="Files in Selected Folders", font=('Arial', 10, 'bold')).pack(anchor=tk.W)
        
        # File tree container frame
        tree_frame = ttk.Frame(file_frame)
        tree_frame.pack(fill=tk.BOTH, expand=True)
        
        # File list with details - updated columns
        columns = ('Name', 'Type', 'Size', 'Modified', 'Relative Path', 'Full Path')
        self.file_tree = ttk.Treeview(tree_frame, columns=columns, show='headings', height=15)
        
        # Configure columns with explicit widths
        self.file_tree.heading('Name', text='File Name', anchor='w')
        self.file_tree.heading('Type', text='Type', anchor='w')
        self.file_tree.heading('Size', text='Size', anchor='w')
        self.file_tree.heading('Modified', text='Modified', anchor='w')
        self.file_tree.heading('Relative Path', text='Folder/Subfolder', anchor='w')
        self.file_tree.heading('Full Path', text='Full Path', anchor='w')
        
        self.file_tree.column('Name', width=180, minwidth=100)
        self.file_tree.column('Type', width=60, minwidth=50)
        self.file_tree.column('Size', width=80, minwidth=60)
        self.file_tree.column('Modified', width=120, minwidth=100)
        self.file_tree.column('Relative Path', width=200, minwidth=150)
        self.file_tree.column('Full Path', width=250, minwidth=200)
        
        # Scrollbars for file tree
        file_scrollbar_v = ttk.Scrollbar(tree_frame, orient=tk.VERTICAL, command=self.file_tree.yview)
        file_scrollbar_h = ttk.Scrollbar(tree_frame, orient=tk.HORIZONTAL, command=self.file_tree.xview)
        
        self.file_tree.configure(yscrollcommand=file_scrollbar_v.set, xscrollcommand=file_scrollbar_h.set)
        
        # Add horizontal scrollbar frame
        scrollbar_frame = ttk.Frame(tree_frame)
        scrollbar_frame.pack(fill=tk.X)
        file_scrollbar_h.pack(fill=tk.X)
        
        # Bind file selection
        self.file_tree.bind('<Double-1>', self.on_file_double_click)
        self.file_tree.bind('<Button-3>', self.on_file_right_click)
        
        # Bottom controls for file operations
        file_controls = ttk.Frame(right_frame)
        file_controls.pack(fill=tk.X, pady=(5, 0))
        
        ttk.Button(file_controls, text="Open File", command=self.open_selected_file).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(file_controls, text="Open Folder", command=self.open_file_location).pack(side=tk.LEFT, padx=(0, 5))
        ttk.Button(file_controls, text="Copy Path", command=self.copy_file_path).pack(side=tk.LEFT, padx=(0, 5))
        
        # Add a test button for debugging
        test_frame = ttk.Frame(file_controls)
        test_frame.pack(side=tk.RIGHT, padx=(10, 0))
        
        ttk.Button(test_frame, text="Test TreeView", command=self.test_treeview).pack(side=tk.LEFT)
        
        # Filter frame
        filter_frame = ttk.Frame(right_frame)
        filter_frame.pack(fill=tk.X, pady=(5, 0))
        
        ttk.Label(filter_frame, text="Filter:").pack(side=tk.LEFT, padx=(0, 5))
        
        self.filter_var = tk.StringVar()
        self.filter_entry = ttk.Entry(filter_frame, textvariable=self.filter_var, width=20)
        self.filter_entry.pack(side=tk.LEFT, padx=(0, 5))
        self.filter_entry.bind('<KeyRelease>', self.filter_files)
        
        # File type filter
        ttk.Label(filter_frame, text="Type:").pack(side=tk.LEFT, padx=(10, 5))
        self.type_filter = ttk.Combobox(filter_frame, values=['All', 'PDF', 'DWG', 'Documents'], width=10)
        self.type_filter.set('All')
        self.type_filter.pack(side=tk.LEFT, padx=(0, 5))
        self.type_filter.bind('<<ComboboxSelected>>', self.filter_files)
        
        # Status bar with progress bar
        status_frame = ttk.Frame(main_frame)
        status_frame.pack(fill=tk.X, pady=(5, 0))
        
        self.status_var = tk.StringVar()
        self.status_var.set("Ready")
        status_label = ttk.Label(status_frame, textvariable=self.status_var, relief=tk.SUNKEN)
        status_label.pack(side=tk.LEFT, fill=tk.X, expand=True)
        
        # Progress bar (initially hidden)
        self.progress_var = tk.DoubleVar()
        self.progress_bar = ttk.Progressbar(
            status_frame, 
            variable=self.progress_var, 
            maximum=100, 
            length=200,
            mode='determinate'
        )
        # Don't pack initially - will be shown when needed
        
        # Context menu for files
        self.context_menu = tk.Menu(self.root, tearoff=0)
        self.context_menu.add_command(label="Open", command=self.open_selected_file)
        self.context_menu.add_command(label="Open Location", command=self.open_file_location)
        self.context_menu.add_separator()
        self.context_menu.add_command(label="Copy Path", command=self.copy_file_path)
        self.context_menu.add_command(label="Properties", command=self.show_file_properties)
        
        # Set default path to current directory
        self.path_var.set(os.getcwd())
    
    def browse_path(self):
        """Open dialog to select a directory"""
        path = filedialog.askdirectory(initialdir=self.path_var.get())
        if path:
            self.path_var.set(path)
    
    def load_folders(self):
        """Load folders from the selected path"""
        path = self.path_var.get()
        if not os.path.exists(path):
            messagebox.showerror("Error", f"Path does not exist: {path}")
            return
        
        self.status_var.set("Loading folders...")
        self.root.update()
        
        # Clear existing items
        self.folder_tree.delete(*self.folder_tree.get_children())
        
        try:
            # Load folders in a separate thread to avoid freezing
            threading.Thread(target=self._load_folders_thread, args=(path,), daemon=True).start()
        except Exception as e:
            messagebox.showerror("Error", f"Failed to load folders: {str(e)}")
            self.status_var.set("Ready")
    
    def _load_folders_thread(self, path):
        """Load folders in a separate thread"""
        try:
            self._populate_tree(path, "")
            self.root.after(0, lambda: self.status_var.set("Folders loaded successfully"))
        except Exception as e:
            self.root.after(0, lambda: messagebox.showerror("Error", f"Failed to load folders: {str(e)}"))
            self.root.after(0, lambda: self.status_var.set("Ready"))
    
    def _populate_tree(self, path, parent):
        """Recursively populate the folder tree"""
        try:
            items = []
            for item in os.listdir(path):
                item_path = os.path.join(path, item)
                if os.path.isdir(item_path):
                    items.append((item, item_path))
            
            # Sort items
            items.sort(key=lambda x: x[0].lower())
            
            for item_name, item_path in items:
                # Add checkbox symbol to folder names
                display_name = f"☐ {item_name}"
                node = self.folder_tree.insert(parent, tk.END, text=display_name, values=(item_path,))
                
                # Check if folder has subfolders
                try:
                    if any(os.path.isdir(os.path.join(item_path, sub)) for sub in os.listdir(item_path)):
                        # Add a dummy child to show the expand icon
                        self.folder_tree.insert(node, tk.END, text="Loading...")
                except (PermissionError, OSError):
                    pass
        except (PermissionError, OSError) as e:
            print(f"Cannot access {path}: {e}")
    
    def on_folder_click(self, event):
        """Handle folder click events"""
        item = self.folder_tree.identify('item', event.x, event.y)
        if item:
            # Get the folder path first
            values = self.folder_tree.item(item, 'values')
            if not values:
                return
            folder_path = values[0]
            
            # Toggle checkbox
            current_text = self.folder_tree.item(item, 'text')
            if current_text.startswith('☐'):
                new_text = current_text.replace('☐', '☑', 1)
                self.folder_tree.item(item, text=new_text)
                # Add to selected folders
                if folder_path not in self.selected_folders:
                    self.selected_folders.append(folder_path)
            elif current_text.startswith('☑'):
                new_text = current_text.replace('☑', '☐', 1)
                self.folder_tree.item(item, text=new_text)
                # Remove from selected folders
                if folder_path in self.selected_folders:
                    self.selected_folders.remove(folder_path)
            
            # Update file list
            self.update_file_list()
    
    def on_folder_double_click(self, event):
        """Handle folder double-click to expand/collapse"""
        item = self.folder_tree.identify('item', event.x, event.y)
        if item:
            # Check if item has dummy children
            children = self.folder_tree.get_children(item)
            if children and self.folder_tree.item(children[0], 'text') == "Loading...":
                # Remove dummy child and load real children
                self.folder_tree.delete(*children)
                folder_path = self.folder_tree.item(item, 'values')[0]
                self._populate_tree(folder_path, item)
    
    def show_progress_bar(self):
        """Show the progress bar"""
        self.progress_bar.pack(side=tk.RIGHT, padx=(10, 0))
        self.progress_var.set(0)
        
    def hide_progress_bar(self):
        """Hide the progress bar"""
        self.progress_bar.pack_forget()
        
    def update_progress(self, value, message=""):
        """Update progress bar and status message"""
        self.progress_var.set(value)
        if message:
            self.status_var.set(message)
        self.root.update_idletasks()

    def scan_folder_recursively(self, folder_path, max_depth=5, current_depth=0):
        """Recursively scan folder and all subfolders for files"""
        files_found = []
        
        if current_depth > max_depth:
            return files_found
            
        try:
            if not os.path.exists(folder_path):
                return files_found
                
            for item in os.listdir(folder_path):
                item_path = os.path.join(folder_path, item)
                
                if os.path.isfile(item_path):
                    # Add file to list
                    file_info = self.get_file_info(item_path)
                    files_found.append(file_info)
                    
                elif os.path.isdir(item_path):
                    # Recursively scan subdirectory
                    try:
                        subfiles = self.scan_folder_recursively(item_path, max_depth, current_depth + 1)
                        files_found.extend(subfiles)
                    except (PermissionError, OSError):
                        continue
                        
        except (PermissionError, OSError) as e:
            print(f"Cannot access {folder_path}: {e}")
        
        return files_found

    def get_scan_depth(self):
        """Get the selected scan depth"""
        depth_text = self.depth_var.get()
        if "Quick" in depth_text:
            return 2
        elif "Medium" in depth_text:
            return 3
        else:  # Deep
            return 5

    def scan_folder_fast(self, folder_path, max_depth=None):
        """Fast folder scanning with configurable depth"""
        if max_depth is None:
            max_depth = self.get_scan_depth()
            
        files_found = []
        
        try:
            if not os.path.exists(folder_path):
                print(f"Path does not exist: {folder_path}")
                return files_found
            
            print(f"Scanning folder: {folder_path} (depth: {max_depth})")
            
            # Scan current folder
            items = os.listdir(folder_path)
            print(f"Found {len(items)} items in {folder_path}")
            
            for item in items:
                item_path = os.path.join(folder_path, item)
                
                if os.path.isfile(item_path):
                    # Only get basic file info for speed
                    try:
                        file_ext = os.path.splitext(item_path)[1].upper().lstrip('.')
                        if not file_ext:
                            file_ext = 'FILE'
                        
                        stat = os.stat(item_path)
                        
                        file_info = {
                            'name': os.path.basename(item_path),
                            'type': file_ext,
                            'size': self.format_file_size(stat.st_size),
                            'modified': datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M"),
                            'path': item_path,
                            'relative_path': os.path.basename(item_path),
                            'raw_size': stat.st_size
                        }
                        
                        files_found.append(file_info)
                        
                    except (OSError, PermissionError) as e:
                        print(f"Error processing file {item_path}: {e}")
                        continue
                
                elif os.path.isdir(item_path) and max_depth > 0:
                    # Recursively scan with reduced depth
                    try:
                        sub_files = self.scan_folder_fast(item_path, max_depth - 1)
                        for sub_file in sub_files:
                            # Update relative path to include parent folder
                            sub_file['relative_path'] = os.path.join(item, sub_file['relative_path'])
                        files_found.extend(sub_files)
                    except (PermissionError, OSError) as e:
                        print(f"Error scanning subfolder {item_path}: {e}")
                        continue
                        
        except (PermissionError, OSError) as e:
            print(f"Cannot access {folder_path}: {e}")
        
        print(f"Found {len(files_found)} files in {folder_path}")
        return files_found

    def update_file_list_threaded(self):
        """Update file list using threading for better performance"""
        if not self.selected_folders:
            self.status_var.set("No folders selected - Click on folder checkboxes to select them")
            return
        
        # Clear existing files
        for item in self.file_tree.get_children():
            self.file_tree.delete(item)
        self.current_files.clear()
        
        # Show progress bar
        self.show_progress_bar()
        self.status_var.set("Starting scan...")
        
        # Start scanning in a separate thread
        def scan_worker():
            try:
                total_files = 0
                all_files = []
                
                for i, folder_path in enumerate(self.selected_folders):
                    folder_name = os.path.basename(folder_path)
                    
                    # Update progress on main thread
                    progress = (i / len(self.selected_folders)) * 100
                    self.root.after(0, lambda p=progress, name=folder_name, idx=i: 
                                  self.update_progress(p, f"Scanning {name}... ({idx+1}/{len(self.selected_folders)})"))
                    
                    # Quick scan with configurable depth
                    folder_files = self.scan_folder_fast(folder_path)
                    all_files.extend(folder_files)
                    total_files += len(folder_files)
                    
                    # Debug output
                    print(f"Scanned {folder_name}: found {len(folder_files)} files")
                
                # Update UI on main thread
                print(f"Total files found: {total_files}")
                self.root.after(0, lambda: self.finish_scan(all_files, total_files))
                
            except Exception as e:
                print(f"Scan error: {e}")
                self.root.after(0, lambda: self.handle_scan_error(str(e)))
        
        # Start the worker thread
        thread = threading.Thread(target=scan_worker, daemon=True)
        thread.start()
    
    def finish_scan(self, all_files, total_files):
        """Finish the scan and update UI"""
        self.current_files = all_files
        
        # Apply filters and display files immediately
        self.filter_files()
        
        # Hide progress bar
        self.hide_progress_bar()
        
        if total_files > 0:
            # Force update the display
            self.root.update_idletasks()
            self.status_var.set(f"Found {total_files} files in {len(self.selected_folders)} folders (quick scan)")
        else:
            self.status_var.set("No files found in selected folders")
    
    def handle_scan_error(self, error_msg):
        """Handle scanning errors"""
        self.hide_progress_bar()
        self.status_var.set(f"Scan error: {error_msg}")
        print(f"Scanning error: {error_msg}")

    def update_file_list_simple(self):
        """Simple, non-threaded file list update for testing"""
        if not self.selected_folders:
            self.status_var.set("No folders selected - Click on folder checkboxes to select them")
            return
        
        # Clear existing files
        for item in self.file_tree.get_children():
            self.file_tree.delete(item)
        self.current_files.clear()
        
        self.status_var.set("Scanning folders...")
        self.root.update()
        
        total_files = 0
        for i, folder_path in enumerate(self.selected_folders):
            folder_name = os.path.basename(folder_path)
            self.status_var.set(f"Scanning {folder_name}... ({i+1}/{len(self.selected_folders)})")
            self.root.update()
            
            folder_files = self.scan_folder_fast(folder_path)
            self.current_files.extend(folder_files)
            total_files += len(folder_files)
            
            print(f"Scanned {folder_name}: found {len(folder_files)} files")
        
        print(f"Total files found: {total_files}")
        print(f"Sample files: {[f['name'] for f in self.current_files[:5]]}")
        
        # Apply filters and display
        self.filter_files()
        
        if total_files > 0:
            self.status_var.set(f"Found {total_files} files in {len(self.selected_folders)} folders")
        else:
            self.status_var.set("No files found in selected folders")

    def update_file_list(self):
        """Update the file list - use simple version for now"""
        self.update_file_list_simple()
    
    def get_file_info(self, file_path):
        """Get file information"""
        try:
            stat = os.stat(file_path)
            size = self.format_file_size(stat.st_size)
            modified = datetime.fromtimestamp(stat.st_mtime).strftime("%Y-%m-%d %H:%M:%S")
            file_ext = os.path.splitext(file_path)[1].upper().lstrip('.')
            
            # Get relative path from base folder for better display
            relative_path = file_path
            for selected_folder in self.selected_folders:
                if file_path.startswith(selected_folder):
                    relative_path = os.path.relpath(file_path, selected_folder)
                    break
            
            return {
                'name': os.path.basename(file_path),
                'type': file_ext if file_ext else 'FILE',
                'size': size,
                'modified': modified,
                'path': file_path,
                'relative_path': relative_path,
                'raw_size': stat.st_size
            }
        except (OSError, PermissionError):
            return {
                'name': os.path.basename(file_path),
                'type': 'Unknown',
                'size': 'Unknown',
                'modified': 'Unknown',
                'path': file_path,
                'relative_path': os.path.basename(file_path),
                'raw_size': 0
            }
    
    def format_file_size(self, size_bytes):
        """Format file size in human readable format"""
        if size_bytes == 0:
            return "0 B"
        
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size_bytes < 1024.0:
                return f"{size_bytes:.1f} {unit}"
            size_bytes /= 1024.0
        return f"{size_bytes:.1f} TB"
    
    def filter_files(self, event=None):
        """Filter files based on search criteria - simplified version"""
        print(f"Starting filter_files with {len(self.current_files)} files...")
        
        # Clear TreeView completely
        self.file_tree.delete(*self.file_tree.get_children())
        
        if not self.current_files:
            print("No files to display")
            return
        
        search_term = self.filter_var.get().lower()
        type_filter = self.type_filter.get()
        
        displayed_count = 0
        
        # Process files with simpler logic
        for i, file_info in enumerate(self.current_files):
            try:
                # Apply filters
                if search_term and search_term not in file_info['name'].lower():
                    continue
                
                if type_filter != 'All':
                    if type_filter == 'PDF' and file_info['type'] != 'PDF':
                        continue
                    elif type_filter == 'DWG' and file_info['type'] != 'DWG':
                        continue
                
                # Create simple tuple of values
                file_values = (
                    file_info.get('name', ''),
                    file_info.get('type', ''),
                    file_info.get('size', ''),
                    file_info.get('modified', ''),
                    file_info.get('relative_path', ''),
                    file_info.get('path', '')
                )
                
                # Insert into TreeView
                item_id = self.file_tree.insert('', 'end', values=file_values)
                displayed_count += 1
                
                # Print progress for first few items
                if i < 3:
                    print(f"Added file {i}: {file_info['name']} -> {item_id}")
                
            except Exception as e:
                print(f"Error processing file {i}: {e}")
                continue
        
        print(f"Filter complete: displayed {displayed_count} files")
        
        # Simple UI refresh
        self.file_tree.update()
        self.root.update()
        
        # Verify TreeView contents
        children = self.file_tree.get_children()
        print(f"TreeView verification: {len(children)} items present")
        
        if children:
            # Make first item visible
            try:
                self.file_tree.see(children[0])
                first_item = self.file_tree.item(children[0])
                print(f"First item check: {first_item['values']}")
            except Exception as e:
                print(f"Error accessing first item: {e}")
        
        # Update status
        self.status_var.set(f"Displayed {displayed_count} of {len(self.current_files)} files")
    
    def on_file_double_click(self, event):
        """Handle file double-click to open"""
        self.open_selected_file()
    
    def on_file_right_click(self, event):
        """Handle right-click on file"""
        item = self.file_tree.identify('item', event.x, event.y)
        if item:
            self.file_tree.selection_set(item)
            self.context_menu.post(event.x_root, event.y_root)
    
    def open_selected_file(self):
        """Open the selected file with default application"""
        selection = self.file_tree.selection()
        if not selection:
            messagebox.showwarning("Warning", "Please select a file to open")
            return
        
        file_path = self.file_tree.item(selection[0], 'values')[5]  # Full Path column
        
        try:
            if sys.platform == "win32":
                os.startfile(file_path)
            elif sys.platform == "darwin":
                subprocess.call(["open", file_path])
            else:
                subprocess.call(["xdg-open", file_path])
        except Exception as e:
            messagebox.showerror("Error", f"Failed to open file: {str(e)}")
    
    def open_file_location(self):
        """Open the folder containing the selected file"""
        selection = self.file_tree.selection()
        if not selection:
            messagebox.showwarning("Warning", "Please select a file")
            return
        
        file_path = self.file_tree.item(selection[0], 'values')[5]  # Full Path column
        folder_path = os.path.dirname(file_path)
        
        try:
            if sys.platform == "win32":
                subprocess.run(['explorer', '/select,', file_path])
            elif sys.platform == "darwin":
                subprocess.call(["open", "-R", file_path])
            else:
                subprocess.call(["xdg-open", folder_path])
        except Exception as e:
            messagebox.showerror("Error", f"Failed to open location: {str(e)}")
    
    def copy_file_path(self):
        """Copy file path to clipboard"""
        selection = self.file_tree.selection()
        if not selection:
            messagebox.showwarning("Warning", "Please select a file")
            return
        
        file_path = self.file_tree.item(selection[0], 'values')[5]  # Full Path column
        self.root.clipboard_clear()
        self.root.clipboard_append(file_path)
        self.status_var.set("File path copied to clipboard")
    
    def show_file_properties(self):
        """Show file properties dialog"""
        selection = self.file_tree.selection()
        if not selection:
            messagebox.showwarning("Warning", "Please select a file")
            return
        
        values = self.file_tree.item(selection[0], 'values')
        file_path = values[5]  # Full Path column
        
        try:
            stat = os.stat(file_path)
            props = f"""File: {values[0]}
Type: {values[1]}
Size: {values[2]}
Modified: {values[3]}
Relative Path: {values[4]}
Full Path: {file_path}
Absolute Path: {os.path.abspath(file_path)}
Readable: {os.access(file_path, os.R_OK)}
Writable: {os.access(file_path, os.W_OK)}"""
            
            messagebox.showinfo("File Properties", props)
        except Exception as e:
            messagebox.showerror("Error", f"Failed to get file properties: {str(e)}")
    
    def test_treeview(self):
        """Test function to add sample data to TreeView"""
        print("Testing TreeView with sample data...")
        
        # Clear existing items
        self.file_tree.delete(*self.file_tree.get_children())
        
        # Add simple, visible test data
        test_files = [
            ("TEST1.PDF", "PDF", "100KB", "2025-01-01", "TestFolder", "C:/test1.pdf"),
            ("TEST2.DWG", "DWG", "200KB", "2025-01-02", "TestFolder", "C:/test2.dwg"),
            ("TEST3.DOC", "DOC", "150KB", "2025-01-03", "TestFolder", "C:/test3.doc"),
            ("SAMPLE.TXT", "TXT", "50KB", "2025-01-04", "TestFolder", "C:/sample.txt"),
            ("EXAMPLE.ZIP", "ZIP", "300KB", "2025-01-05", "TestFolder", "C:/example.zip")
        ]
        
        print(f"Inserting {len(test_files)} test files...")
        
        for i, file_data in enumerate(test_files):
            try:
                item = self.file_tree.insert('', 'end', values=file_data)
                print(f"Test file {i+1} inserted: {file_data[0]} -> ID: {item}")
            except Exception as e:
                print(f"Error inserting test file {i+1}: {e}")
        
        # Force TreeView to update and be visible
        self.file_tree.update_idletasks()
        self.file_tree.update()
        self.root.update_idletasks()
        self.root.update()
        
        # Check results
        children = self.file_tree.get_children()
        print(f"TreeView now contains {len(children)} test items")
        print(f"TreeView widget info: {self.file_tree.winfo_width()}x{self.file_tree.winfo_height()}")
        
        if children:
            # Try to make first item visible and selected
            self.file_tree.see(children[0])
            self.file_tree.selection_set(children[0])
            self.file_tree.focus(children[0])
            
            values = self.file_tree.item(children[0])['values']
            print(f"First item values: {values}")
            
            # Try to scroll to make sure items are visible
            self.file_tree.yview_moveto(0.0)
        
        self.status_var.set(f"TEST: {len(children)} test files added - Check if visible in GUI!")
        
        # Force focus on TreeView
        self.file_tree.focus_set()

    def refresh_view(self):
        """Refresh the current view"""
        if self.selected_folders:
            self.update_file_list()
        else:
            self.load_folders()
    
    def select_all_folders(self):
        """Select all visible folders - faster version"""
        self.selected_folders.clear()
        
        # Quick selection without progress bar for speed
        def select_recursive(parent):
            for item in self.folder_tree.get_children(parent):
                current_text = self.folder_tree.item(item, 'text')
                if current_text.startswith('☐'):
                    # Change to checked
                    new_text = current_text.replace('☐', '☑', 1)
                    self.folder_tree.item(item, text=new_text)
                    # Add to selected folders
                    values = self.folder_tree.item(item, 'values')
                    if values:
                        folder_path = values[0]
                        if folder_path not in self.selected_folders:
                            self.selected_folders.append(folder_path)
                
                # Recursively select children
                select_recursive(item)
        
        self.status_var.set("Selecting all folders...")
        select_recursive("")
        self.update_file_list()
    
    def _get_all_folder_items(self):
        """Generator to get all folder items for counting"""
        def get_items_recursive(parent):
            for item in self.folder_tree.get_children(parent):
                yield item
                yield from get_items_recursive(item)
        
        yield from get_items_recursive("")
    
    def clear_selection(self):
        """Clear all folder selections"""
        self.selected_folders.clear()
        
        def clear_recursive(parent):
            for item in self.folder_tree.get_children(parent):
                current_text = self.folder_tree.item(item, 'text')
                if current_text.startswith('☑'):
                    # Change to unchecked
                    new_text = current_text.replace('☑', '☐', 1)
                    self.folder_tree.item(item, text=new_text)
                
                # Recursively clear children
                clear_recursive(item)
        
        clear_recursive("")
        
        # Clear file list
        self.file_tree.delete(*self.file_tree.get_children())
        self.current_files.clear()
        self.status_var.set("All selections cleared")

def main():
    """Main function to run the application"""
    root = tk.Tk()
    
    # Set application icon (if available)
    try:
        # You can add an icon file here
        # root.iconbitmap('icon.ico')
        pass
    except:
        pass
    
    app = FileBrowserGUI(root)
    
    # Center the window
    root.update_idletasks()
    width = root.winfo_width()
    height = root.winfo_height()
    x = (root.winfo_screenwidth() // 2) - (width // 2)
    y = (root.winfo_screenheight() // 2) - (height // 2)
    root.geometry(f"{width}x{height}+{x}+{y}")
    
    root.mainloop()

if __name__ == "__main__":
    main()
