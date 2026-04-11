# tree-file-system
#ifndef FILE_SYSTEM_H
#define FILE_SYSTEM_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>

#define MAX_NAME_LEN 256
#define MAX_PATH_LEN 1024

typedef enum {
    NODE_FILE,
    NODE_DIR
} NodeType;

typedef struct FileNode {
    char name[MAX_NAME_LEN];
    NodeType type;
    struct FileNode **children;
    int child_count;
    int child_capacity;
    struct FileNode *parent;
} FileNode;

typedef struct {
    FileNode *root;
} FileSystem;

// Initialization
FileSystem* fs_create();
void fs_destroy(FileNode *node);

// Directory operations
bool fs_mkdir(FileNode *current, const char *dirname);
FileNode* fs_cd(FileNode *current, const char *dirname);
FileNode* fs_find_child(FileNode *current, const char *name);

// File operations
bool fs_touch(FileNode *current, const char *filename);

// Display operations
void fs_list_dfs(FileNode *node, int depth);
void fs_list_bfs(FileNode *root);
void fs_print_tree(FileNode *node, int depth, bool is_last);

// Search operations (BST ordering)
FileNode* fs_search_bst(FileNode *node, const char *name, int *comparisons);
void fs_search_all(FileNode *node, const char *name);

// Utility
void fs_print_full_path(FileNode *node);
FileNode* fs_get_parent(FileNode *node);

#endif // FILE_SYSTEM_H
#include "file_system.h"

// Initialize file system
FileSystem* fs_create() {
    FileSystem *fs = (FileSystem*)malloc(sizeof(FileSystem));
    fs->root = (FileNode*)malloc(sizeof(FileNode));
    strcpy(fs->root->name, "/");
    fs->root->type = NODE_DIR;
    fs->root->children = (FileNode**)malloc(10 * sizeof(FileNode*));
    fs->root->child_count = 0;
    fs->root->child_capacity = 10;
    fs->root->parent = NULL;
    return fs;
}

// Destroy file system recursively
void fs_destroy(FileNode *node) {
    if (!node) return;
    
    for (int i = 0; i < node->child_count; i++) {
        fs_destroy(node->children[i]);
    }
    
    free(node->children);
    free(node);
}

// Expand children array when needed
static void expand_children(FileNode *node) {
    if (node->child_count >= node->child_capacity) {
        node->child_capacity *= 2;
        node->children = (FileNode**)realloc(node->children, 
                                            node->child_capacity * sizeof(FileNode*));
    }
}

// Compare function for BST ordering (alphabetical)
static int compare_names(const char *name1, const char *name2) {
    return strcmp(name1, name2);
}

// Find child node by name (linear search)
FileNode* fs_find_child(FileNode *current, const char *name) {
    if (!current || current->type != NODE_DIR) return NULL;
    
    for (int i = 0; i < current->child_count; i++) {
        if (strcmp(current->children[i]->name, name) == 0) {
            return current->children[i];
        }
    }
    return NULL;
}

// Create a new directory
bool fs_mkdir(FileNode *current, const char *dirname) {
    if (!current || current->type != NODE_DIR) {
        printf("Error: Cannot create directory in a file\n");
        return false;
    }
    
    if (fs_find_child(current, dirname)) {
        printf("Error: Directory '%s' already exists\n", dirname);
        return false;
    }
    
    expand_children(current);
    
    FileNode *new_dir = (FileNode*)malloc(sizeof(FileNode));
    strcpy(new_dir->name, dirname);
    new_dir->type = NODE_DIR;
    new_dir->children = (FileNode**)malloc(10 * sizeof(FileNode*));
    new_dir->child_count = 0;
    new_dir->child_capacity = 10;
    new_dir->parent = current;
    
    // Insert in sorted order (BST)
    int insert_pos = current->child_count;
    for (int i = 0; i < current->child_count; i++) {
        if (compare_names(dirname, current->children[i]->name) < 0) {
            insert_pos = i;
            break;
        }
    }
    
    // Shift elements to make room
    for (int i = current->child_count; i > insert_pos; i--) {
        current->children[i] = current->children[i-1];
    }
    
    current->children[insert_pos] = new_dir;
    current->child_count++;
    
    printf("Directory '%s' created successfully\n", dirname);
    return true;
}

// Create a new file
bool fs_touch(FileNode *current, const char *filename) {
    if (!current || current->type != NODE_DIR) {
        printf("Error: Cannot create file in a file\n");
        return false;
    }
    
    if (fs_find_child(current, filename)) {
        printf("Error: File '%s' already exists\n", filename);
        return false;
    }
    
    expand_children(current);
    
    FileNode *new_file = (FileNode*)malloc(sizeof(FileNode));
    strcpy(new_file->name, filename);
    new_file->type = NODE_FILE;
    new_file->children = NULL;
    new_file->child_count = 0;
    new_file->child_capacity = 0;
    new_file->parent = current;
    
    // Insert in sorted order (BST)
    int insert_pos = current->child_count;
    for (int i = 0; i < current->child_count; i++) {
        if (compare_names(filename, current->children[i]->name) < 0) {
            insert_pos = i;
            break;
        }
    }
    
    // Shift elements to make room
    for (int i = current->child_count; i > insert_pos; i--) {
        current->children[i] = current->children[i-1];
    }
    
    current->children[insert_pos] = new_file;
    current->child_count++;
    
    printf("File '%s' created successfully\n", filename);
    return true;
}

// Change directory
FileNode* fs_cd(FileNode *current, const char *dirname) {
    if (!current) return NULL;
    
    if (strcmp(dirname, "..") == 0) {
        return current->parent ? current->parent : current;
    }
    
    if (strcmp(dirname, "/") == 0) {
        FileNode *node = current;
        while (node->parent) {
            node = node->parent;
        }
        return node;
    }
    
    FileNode *child = fs_find_child(current, dirname);
    if (!child) {
        printf("Error: Directory '%s' not found\n", dirname);
        return current;
    }
    
    if (child->type != NODE_DIR) {
        printf("Error: '%s' is not a directory\n", dirname);
        return current;
    }
    
    return child;
}

// DFS traversal for listing
void fs_list_dfs(FileNode *node, int depth) {
    if (!node) return;
    
    for (int i = 0; i < depth; i++) printf("  ");
    
    if (node->type == NODE_DIR) {
        printf("📁 %s/\n", node->name);
    } else {
        printf("📄 %s\n", node->name);
    }
    
    if (node->type == NODE_DIR) {
        for (int i = 0; i < node->child_count; i++) {
            fs_list_dfs(node->children[i], depth + 1);
        }
    }
}

// BFS traversal for listing
void fs_list_bfs(FileNode *root) {
    if (!root) return;
    
    // Simple queue implementation using array
    FileNode **queue = (FileNode**)malloc(1000 * sizeof(FileNode*));
    int front = 0, rear = 0;
    
    queue[rear++] = root;
    int depth_array[1000] = {0};
    
    int idx = 0;
    depth_array[idx] = 0;
    
    while (front < rear) {
        FileNode *current = queue[front];
        int current_depth = depth_array[front];
        front++;
        
        for (int i = 0; i < current_depth; i++) printf("  ");
        
        if (current->type == NODE_DIR) {
            printf("📁 %s/\n", current->name);
        } else {
            printf("📄 %s\n", current->name);
        }
        
        if (current->type == NODE_DIR) {
            for (int i = 0; i < current->child_count; i++) {
                queue[rear] = current->children[i];
                depth_array[rear] = current_depth + 1;
                rear++;
            }
        }
    }
    
    free(queue);
}

// Print full directory tree with indentation
void fs_print_tree(FileNode *node, int depth, bool is_last) {
    if (!node) return;
    
    for (int i = 0; i < depth; i++) {
        if (i == depth - 1) {
            printf(is_last ? "��── " : "├── ");
        } else {
            printf("│   ");
        }
    }
    
    if (depth > 0) {
        if (node->type == NODE_DIR) {
            printf("📁 %s/\n", node->name);
        } else {
            printf("📄 %s\n", node->name);
        }
    } else {
        printf("📁 %s/\n", node->name);
    }
    
    if (node->type == NODE_DIR) {
        for (int i = 0; i < node->child_count; i++) {
            bool is_last_child = (i == node->child_count - 1);
            fs_print_tree(node->children[i], depth + 1, is_last_child);
        }
    }
}

// Search using BST ordering (binary search on sorted children)
FileNode* fs_search_bst(FileNode *node, const char *name, int *comparisons) {
    if (!node) return NULL;
    
    *comparisons = 0;
    
    if (node->type == NODE_DIR) {
        int left = 0, right = node->child_count - 1;
        
        while (left <= right) {
            int mid = (left + right) / 2;
            int cmp = compare_names(name, node->children[mid]->name);
            (*comparisons)++;
            
            if (cmp == 0) {
                return node->children[mid];
            } else if (cmp < 0) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
    }
    
    return NULL;
}

// Search all occurrences recursively
void fs_search_all(FileNode *node, const char *name) {
    if (!node) return;
    
    if (strcmp(node->name, name) == 0) {
        printf("Found: ");
        fs_print_full_path(node);
        printf("\n");
    }
    
    if (node->type == NODE_DIR) {
        for (int i = 0; i < node->child_count; i++) {
            fs_search_all(node->children[i], name);
        }
    }
}

// Print full path of a node
void fs_print_full_path(FileNode *node) {
    if (!node) return;
    
    if (node->parent) {
        fs_print_full_path(node->parent);
        printf("/%s", node->name);
    } else {
        printf("/");
    }
}

// Get parent directory
FileNode* fs_get_parent(FileNode *node) {
    return node ? node->parent : NULL;
}
#include "file_system.h"
#include <ctype.h>

#define MAX_COMMAND_LEN 512

typedef struct {
    FileNode *current_dir;
    FileSystem *fs;
} Shell;

void print_prompt(FileNode *node) {
    printf("fs");
    FileNode *temp = node;
    int depth = 0;
    
    while (temp->parent) {
        temp = temp->parent;
        depth++;
    }
    
    if (depth == 0) {
        printf(":/# ");
    } else {
        printf(":~");
        temp = node;
        while (temp->parent && temp->parent->parent) {
            temp = temp->parent;
        }
        printf("/%s# ", temp->name);
    }
    fflush(stdout);
}

void print_help() {
    printf("\n========== FILE SYSTEM COMMANDS ==========\n");
    printf("mkdir <dirname>    - Create a directory\n");
    printf("touch <filename>   - Create a file\n");
    printf("cd <dirname>       - Change directory (use '..' to go up, '/' for root)\n");
    printf("ls                 - List files (DFS traversal)\n");
    printf("ls-bfs             - List files (BFS traversal)\n");
    printf("tree               - Display full directory tree\n");
    printf("search <name>      - Search for file/directory\n");
    printf("search-bst <name>  - Search using BST ordering (in current dir)\n");
    printf("pwd                - Print working directory\n");
    printf("help               - Show this help\n");
    printf("exit               - Exit the shell\n");
    printf("==========================================\n\n");
}

int main() {
    FileSystem *fs = fs_create();
    Shell shell;
    shell.fs = fs;
    shell.current_dir = fs->root;
    
    char command[MAX_COMMAND_LEN];
    
    printf("\n╔════════════════════════════════════════╗\n");
    printf("║   TREE-BASED FILE SYSTEM IN C          ║\n");
    printf("║   Type 'help' for available commands   ║\n");
    printf("╚════════════════════════════════════════╝\n\n");
    
    while (true) {
        print_prompt(shell.current_dir);
        
        if (!fgets(command, sizeof(command), stdin)) {
            break;
        }
        
        // Remove trailing newline
        command[strcspn(command, "\n")] = '\0';
        
        // Skip empty commands
        if (strlen(command) == 0) continue;
        
        // Parse command
        char *cmd = strtok(command, " ");
        if (!cmd) continue;
        
        if (strcmp(cmd, "exit") == 0) {
            printf("Exiting file system...\n");
            break;
        }
        else if (strcmp(cmd, "help") == 0) {
            print_help();
        }
        else if (strcmp(cmd, "mkdir") == 0) {
            char *dirname = strtok(NULL, " ");
            if (!dirname) {
                printf("Usage: mkdir <dirname>\n");
            } else {
                fs_mkdir(shell.current_dir, dirname);
            }
        }
        else if (strcmp(cmd, "touch") == 0) {
            char *filename = strtok(NULL, " ");
            if (!filename) {
                printf("Usage: touch <filename>\n");
            } else {
                fs_touch(shell.current_dir, filename);
            }
        }
        else if (strcmp(cmd, "cd") == 0) {
            char *dirname = strtok(NULL, " ");
            if (!dirname) {
                printf("Usage: cd <dirname>\n");
            } else {
                shell.current_dir = fs_cd(shell.current_dir, dirname);
            }
        }
        else if (strcmp(cmd, "ls") == 0) {
            printf("\n--- DFS Traversal ---\n");
            if (shell.current_dir->child_count == 0) {
                printf("(empty directory)\n");
            } else {
                for (int i = 0; i < shell.current_dir->child_count; i++) {
                    fs_list_dfs(shell.current_dir->children[i], 0);
                }
            }
            printf("\n");
        }
        else if (strcmp(cmd, "ls-bfs") == 0) {
            printf("\n--- BFS Traversal ---\n");
            if (shell.current_dir->child_count == 0) {
                printf("(empty directory)\n");
            } else {
                for (int i = 0; i < shell.current_dir->child_count; i++) {
                    fs_list_bfs(shell.current_dir->children[i]);
                }
            }
            printf("\n");
        }
        else if (strcmp(cmd, "tree") == 0) {
            printf("\n--- Full Directory Tree ---\n");
            fs_print_tree(shell.current_dir, 0, true);
            printf("\n");
        }
        else if (strcmp(cmd, "pwd") == 0) {
            printf("Path: ");
            fs_print_full_path(shell.current_dir);
            printf("\n\n");
        }
        else if (strcmp(cmd, "search") == 0) {
            char *name = strtok(NULL, " ");
            if (!name) {
                printf("Usage: search <name>\n");
            } else {
                printf("\nSearching for '%s'...\n", name);
                FileNode *root = shell.current_dir;
                while (root->parent) root = root->parent;
                fs_search_all(root, name);
                printf("\n");
            }
        }
        else if (strcmp(cmd, "search-bst") == 0) {
            char *name = strtok(NULL, " ");
            if (!name) {
                printf("Usage: search-bst <name>\n");
            } else {
                int comparisons = 0;
                FileNode *result = fs_search_bst(shell.current_dir, name, &comparisons);
                printf("\nBST Search for '%s':\n", name);
                printf("Comparisons: %d\n", comparisons);
                if (result) {
                    printf("Found: %s (%s)\n", result->name, 
                           result->type == NODE_DIR ? "directory" : "file");
                } else {
                    printf("Not found in current directory\n");
                }
                printf("\n");
            }
        }
        else {
            printf("Unknown command: '%s'. Type 'help' for available commands.\n\n", cmd);
        }
    }
    
    // Cleanup
    fs_destroy(fs->root);
    free(fs);
    
    return 0;
}
