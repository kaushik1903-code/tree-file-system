# tree-file-system
/*
 * Tree-Based File System Simulator in C
 * =======================================
 * Structures:
 *   - N-ary tree   : filesystem hierarchy
 *   - BST          : name-indexed search
 * Features:
 *   - mkdir, touch, ls, cd, pwd
 *   - DFS / BFS traversal of current directory
 *   - BST search (O(log n) average)
 *   - Pretty tree display with connectors
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_NAME  128
#define MAX_PATH  1024
#define MAX_Q     2048

typedef enum { TYPE_DIR, TYPE_FILE } NodeType;

/* ─── N-ary tree ──────────────────────────────────────────────── */
typedef struct FSNode {
    char         name[MAX_NAME];
    NodeType     type;
    struct FSNode *parent;
    struct FSNode *first_child;
    struct FSNode *next_sibling;
} FSNode;

/* ─── BST ─────────────────────────────────────────────────────── */
typedef struct BSTNode {
    FSNode       *fs;
    struct BSTNode *left, *right;
} BSTNode;

/* ─── Globals ─────────────────────────────────────────────────── */
static FSNode  *root;
static FSNode  *cwd;
static BSTNode *bst_root;

/* ══ BST ══════════════════════════════════════════════════════════ */

static BSTNode *bst_new(FSNode *fs) {
    BSTNode *n = malloc(sizeof *n);
    n->fs = fs; n->left = n->right = NULL;
    return n;
}

static BSTNode *bst_insert(BSTNode *node, FSNode *fs) {
    if (!node) return bst_new(fs);
    int c = strcmp(fs->name, node->fs->name);
    if      (c < 0) node->left  = bst_insert(node->left,  fs);
    else if (c > 0) node->right = bst_insert(node->right, fs);
    /* equal name → go right subtree so both survive */
    else            node->right = bst_insert(node->right, fs);
    return node;
}

static void bst_search(BSTNode *node, const char *name,
                        FSNode **out, int *cnt) {
    if (!node) return;
    int c = strcmp(name, node->fs->name);
    if (c == 0) {
        out[(*cnt)++] = node->fs;
        bst_search(node->left,  name, out, cnt);
        bst_search(node->right, name, out, cnt);
    } else if (c < 0) {
        bst_search(node->left,  name, out, cnt);
    } else {
        bst_search(node->right, name, out, cnt);
    }
}

static void bst_free(BSTNode *n) {
    if (!n) return;
    bst_free(n->left); bst_free(n->right); free(n);
}

/* ══ Filesystem helpers ═══════════════════════════════════════════ */

static FSNode *fs_new(const char *name, NodeType t, FSNode *parent) {
    FSNode *n = malloc(sizeof *n);
    strncpy(n->name, name, MAX_NAME-1); n->name[MAX_NAME-1] = '\0';
    n->type = t; n->parent = parent;
    n->first_child = n->next_sibling = NULL;
    return n;
}

/* Insert child keeping siblings alphabetically sorted */
static void fs_add_child(FSNode *parent, FSNode *child) {
    child->parent = parent;
    if (!parent->first_child ||
        strcmp(child->name, parent->first_child->name) < 0) {
        child->next_sibling = parent->first_child;
        parent->first_child = child;
        return;
    }
    FSNode *p = parent->first_child;
    while (p->next_sibling &&
           strcmp(child->name, p->next_sibling->name) > 0)
        p = p->next_sibling;
    child->next_sibling = p->next_sibling;
    p->next_sibling = child;
}

static FSNode *fs_child(FSNode *parent, const char *name) {
    FSNode *c = parent->first_child;
    while (c) { if (!strcmp(c->name, name)) return c; c = c->next_sibling; }
    return NULL;
}

static void fs_path(FSNode *n, char *buf, size_t sz) {
    if (!n->parent) { strncpy(buf, "/", sz); return; }
    char tmp[MAX_PATH];
    fs_path(n->parent, tmp, sizeof tmp);
    if (!strcmp(tmp, "/")) snprintf(buf, sz, "/%s", n->name);
    else                   snprintf(buf, sz, "%s/%s", tmp, n->name);
}

static void fs_free(FSNode *n) {
    if (!n) return;
    fs_free(n->first_child); fs_free(n->next_sibling); free(n);
}

/* ══ DFS (pre-order, subtree only – no sibling walk at top) ════════ */

/* Recurse only into first_child; siblings are iterated by the caller */
static void dfs_subtree(FSNode *n, int depth) {
    if (!n) return;
    for (int i = 0; i < depth*2; i++) putchar(' ');
    printf("%-7s %s\n",
           n->type == TYPE_DIR ? "[DIR]" : "[FILE]",
           n->name);
    FSNode *child = n->first_child;
    while (child) {
        dfs_subtree(child, depth+1);
        child = child->next_sibling;
    }
}

static void cmd_dfs(FSNode *dir) {
    printf("\n=== DFS traversal of [%s] ===\n", dir->name[0] ? dir->name : "/");
    FSNode *c = dir->first_child;
    if (!c) { puts("  (empty)\n"); return; }
    while (c) {
        dfs_subtree(c, 1);
        c = c->next_sibling;
    }
    putchar('\n');
}

/* ══ BFS ══════════════════════════════════════════════════════════ */

static void cmd_bfs(FSNode *dir) {
    printf("\n=== BFS traversal of [%s] ===\n", dir->name[0] ? dir->name : "/");
    FSNode *q[MAX_Q];
    int head = 0, tail = 0, level_end = 0, level = 1;

    FSNode *c = dir->first_child;
    while (c) { q[tail++] = c; c = c->next_sibling; }
    if (head == tail) { puts("  (empty)\n"); return; }

    level_end = tail;
    while (head < tail) {
        if (head == level_end) {
            level++;
            level_end = tail;
        }
        FSNode *cur = q[head++];
        printf("  L%-2d  %-7s %s\n",
               level,
               cur->type == TYPE_DIR ? "[DIR]" : "[FILE]",
               cur->name);
        c = cur->first_child;
        while (c && tail < MAX_Q) { q[tail++] = c; c = c->next_sibling; }
    }
    putchar('\n');
}

/* ══ Pretty tree ══════════════════════════════════════════════════ */

static void print_tree(FSNode *n, const char *prefix, int is_last) {
    if (!n) return;
    printf("%s%s%s%s\n",
           prefix,
           is_last ? "\\-- " : "|-- ",
           n->name,
           n->type == TYPE_DIR ? "/" : "");
    char np[MAX_PATH];
    snprintf(np, sizeof np, "%s%s", prefix, is_last ? "    " : "|   ");
    FSNode *child = n->first_child;
    while (child) {
        print_tree(child, np, child->next_sibling == NULL);
        child = child->next_sibling;
    }
}

static void cmd_tree(void) {
    puts("\n/");
    FSNode *child = root->first_child;
    while (child) {
        print_tree(child, "", child->next_sibling == NULL);
        child = child->next_sibling;
    }
    putchar('\n');
}

/* ══ User commands ════════════════════════════════════════════════ */

static void cmd_mkdir(const char *name) {
    if (fs_child(cwd, name)) { printf("Error: '%s' already exists.\n", name); return; }
    FSNode *d = fs_new(name, TYPE_DIR, cwd);
    fs_add_child(cwd, d);
    bst_root = bst_insert(bst_root, d);
    printf("Directory '%s' created.\n", name);
}

static void cmd_touch(const char *name) {
    if (fs_child(cwd, name)) { printf("Error: '%s' already exists.\n", name); return; }
    FSNode *f = fs_new(name, TYPE_FILE, cwd);
    fs_add_child(cwd, f);
    bst_root = bst_insert(bst_root, f);
    printf("File '%s' created.\n", name);
}

static void cmd_ls(void) {
    printf("\nContents of [%s]:\n", cwd->name[0] ? cwd->name : "/");
    FSNode *c = cwd->first_child;
    if (!c) { puts("  (empty)\n"); return; }
    while (c) {
        printf("  %-7s %s\n",
               c->type == TYPE_DIR ? "[DIR]" : "[FILE]",
               c->name);
        c = c->next_sibling;
    }
    putchar('\n');
}

static void cmd_cd(const char *name) {
    if (!strcmp(name, ".."))  { if (cwd->parent) cwd = cwd->parent; else puts("Already at root."); return; }
    if (!strcmp(name, "/"))   { cwd = root; return; }
    FSNode *t = fs_child(cwd, name);
    if (!t)                    { printf("Error: '%s' not found.\n",     name); return; }
    if (t->type == TYPE_FILE)  { printf("Error: '%s' is a file.\n",     name); return; }
    cwd = t;
}

static void cmd_pwd(void) {
    char path[MAX_PATH];
    fs_path(cwd, path, sizeof path);
    if (!path[0]) strcpy(path, "/");
    printf("%s\n", path);
}

static void cmd_search(const char *name) {
    FSNode *res[MAX_Q]; int cnt = 0;
    bst_search(bst_root, name, res, &cnt);
    if (!cnt) { printf("Not found: '%s'\n", name); return; }
    printf("\nBST search results for '%s':\n", name);
    for (int i = 0; i < cnt; i++) {
        char path[MAX_PATH];
        fs_path(res[i], path, sizeof path);
        printf("  %-7s %s\n",
               res[i]->type == TYPE_DIR ? "[DIR]" : "[FILE]",
               path);
    }
    putchar('\n');
}

/* ══ Seed helpers (silent) ════════════════════════════════════════ */
static void smkdir(const char *n) {
    FSNode *d = fs_new(n,TYPE_DIR,cwd); fs_add_child(cwd,d);
    bst_root = bst_insert(bst_root,d);
}
static void stouch(const char *n) {
    FSNode *f = fs_new(n,TYPE_FILE,cwd); fs_add_child(cwd,f);
    bst_root = bst_insert(bst_root,f);
}
static void scd(const char *n) {
    if (!strcmp(n,"..")) { if (cwd->parent) cwd=cwd->parent; return; }
    FSNode *t = fs_child(cwd,n);
    if (t && t->type==TYPE_DIR) cwd=t;
}

static void seed(void) {
    smkdir("home");         scd("home");
      smkdir("alice");      scd("alice");
        stouch("notes.txt");
        stouch("resume.pdf");
        smkdir("projects"); scd("projects");
          stouch("Makefile");
          stouch("main.c");
          smkdir("libs");   scd("libs");
            stouch("utils.c");
            stouch("utils.h");
          scd("..");
        scd("..");
      scd("..");
      smkdir("bob");        scd("bob");
        stouch("photo.png");
        smkdir("docs");     scd("docs");
          stouch("readme.md");
        scd("..");
      scd("..");
    scd("..");
    smkdir("etc");          scd("etc");
      stouch("hosts");
      stouch("passwd");
      smkdir("nginx");      scd("nginx");
        stouch("mime.types");
        stouch("nginx.conf");
      scd("..");
    scd("..");
    smkdir("var");          scd("var");
      smkdir("log");        scd("log");
        stouch("auth.log");
        stouch("syslog");
      scd("..");
    scd("..");
    cwd = root;
}

/* ══ Help & main loop ═════════════════════════════════════════════ */

static void help(void) {
    puts("\nAvailable commands:");
    puts("  mkdir <name>       Create directory in current location");
    puts("  touch <name>       Create file in current location");
    puts("  ls                 List current directory");
    puts("  cd <name|..|/>     Change directory");
    puts("  pwd                Show current path");
    puts("  dfs                DFS traversal from current directory");
    puts("  bfs                BFS traversal from current directory");
    puts("  tree               Full tree from filesystem root");
    puts("  search <name>      BST-based search by name");
    puts("  help               Show this help");
    puts("  exit               Quit\n");
}

static char *trim(char *s) {
    while (isspace((unsigned char)*s)) s++;
    char *e = s + strlen(s) - 1;
    while (e > s && isspace((unsigned char)*e)) *e-- = '\0';
    return s;
}

int main(void) {
    root = fs_new("", TYPE_DIR, NULL);
    cwd  = root; bst_root = NULL;

    puts("=========================================");
    puts("  Tree-Based File System Simulator (C)  ");
    puts("=========================================");
    puts("Seeding sample filesystem...");
    seed();
    puts("Ready. Type 'help' for commands.\n");

    char line[512];
    while (1) {
        char path[MAX_PATH];
        fs_path(cwd, path, sizeof path);
        if (!path[0]) strcpy(path, "/");
        printf("%s $ ", path);
        fflush(stdout);

        if (!fgets(line, sizeof line, stdin)) break;
        char *in = trim(line);
        if (!in[0]) continue;

        char cmd[64]={0}, arg[MAX_NAME]={0};
        sscanf(in, "%63s %127[^\n]", cmd, arg);
        /* trim arg too */
        char *a = trim(arg);

        if      (!strcmp(cmd,"exit"))   break;
        else if (!strcmp(cmd,"help"))   help();
        else if (!strcmp(cmd,"pwd"))    cmd_pwd();
        else if (!strcmp(cmd,"ls"))     cmd_ls();
        else if (!strcmp(cmd,"tree"))   cmd_tree();
        else if (!strcmp(cmd,"dfs"))    cmd_dfs(cwd);
        else if (!strcmp(cmd,"bfs"))    cmd_bfs(cwd);
        else if (!strcmp(cmd,"mkdir"))  { if(*a) cmd_mkdir(a); else puts("Usage: mkdir <name>"); }
        else if (!strcmp(cmd,"touch"))  { if(*a) cmd_touch(a); else puts("Usage: touch <name>"); }
        else if (!strcmp(cmd,"cd"))     { if(*a) cmd_cd(a);    else puts("Usage: cd <name|..|/>"); }
        else if (!strcmp(cmd,"search")) { if(*a) cmd_search(a);else puts("Usage: search <name>"); }
        else printf("Unknown command: '%s'. Type 'help'.\n", cmd);
    }

    fs_free(root);
    bst_free(bst_root);
    puts("Goodbye.");
    return 0;
}
