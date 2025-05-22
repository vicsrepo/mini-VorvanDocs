# mini-VorvanDocs
mini


Řešení využívající **čistý JavaScript s dynamickým načítáním adresářové struktury** bez nutnosti buildování.
Funguje na jakémkoli serveru s povoleným directory listingem:

```html
<!DOCTYPE html>
<html lang="cs">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Live Docs</title>
   <style>
    :root {
      --text: #333;
      --bg: #fff;
      --sidebar-bg: #f8f9fa;
      --border: #dee2e6;
      --hover-bg: rgba(0,0,0,0.05);
    }

    @media (prefers-color-scheme: dark) {
      :root {
        --text: #e9ecef;
        --bg: #212529;
        --sidebar-bg: #2b3035;
        --border: #495057;
        --hover-bg: rgba(255,255,255,0.05);
      }
    }

    body {
      margin: 0;
      padding: 0;
      color: var(--text);
      background: var(--bg);
      font-family: system-ui, -apple-system, sans-serif;
      line-height: 1.6;
    }

    .container {
      display: grid;
      grid-template-columns: minmax(250px, 300px) 1fr;
      min-height: 100vh;
    }

    #sidebar {
      background: var(--sidebar-bg);
      padding: 1rem;
      border-right: 1px solid var(--border);
      overflow-y: auto;
      height: 100vh;
      position: sticky;
      top: 0;
    }

    #content {
      padding: 2rem;
      max-width: 800px;
      margin: 0 auto;
      width: 100%;
      box-sizing: border-box;
    }

    .tree {
      list-style: none;
      padding-left: 0;
      margin: 0;
    }

    .tree-item {
      position: relative;
      padding-left: 1.5rem;
    }

    .tree-item__toggle {
      position: absolute;
      left: 0;
      top: 0.3rem;
      cursor: pointer;
      width: 1rem;
      height: 1rem;
      display: flex;
      align-items: center;
      justify-content: center;
      user-select: none;
    }

    .tree-item__content {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      padding: 0.25rem 0;
      text-decoration: none;
      color: inherit;
      border-radius: 4px;
    }

    .tree-item__content:hover {
      background: var(--hover-bg);
    }

    .tree-item__children {
      list-style: none;
      padding-left: 1rem;
      margin: 0;
      display: none;
    }

    .tree-item--open > .tree-item__children {
      display: block;
    }

    .tree-item--file::before {
      content: "📄";
      margin-right: 0.3rem;
    }

    .tree-item--dir::before {
      content: "📁";
      margin-right: 0.3rem;
    }

    .active {
      font-weight: 600;
      color: #0d6efd;
    }

    pre {
      background: rgba(0,0,0,0.05);
      padding: 1rem;
      border-radius: 6px;
      overflow-x: auto;
    }

    @media (max-width: 768px) {
      .container {
        grid-template-columns: 1fr;
      }

      #sidebar {
        height: auto;
        position: static;
        border-right: none;
        border-bottom: 1px solid var(--border);
      }
    }
  </style>
</head>
<body>
  <div class="container">
    <nav id="sidebar">
      <ul class="tree" id="tree"></ul>
    </nav>

    <main id="content"></main>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
  <script>
    class LiveDocLoader {
      static async getDirStructure(path = '') {
        try {
          const response = await fetch(`docs/${path}`);
          const html = await response.text();
          const parser = new DOMParser();
          const doc = parser.parseFromString(html, 'text/html');

          return Array.from(doc.querySelectorAll('a'))
            .filter(a => a.href.includes('/docs/'))
            .map(a => {
              const fullPath = new URL(a.href).pathname;
              const itemPath = fullPath.split('/docs/')[1] || '';
              return {
                name: a.textContent.replace(/\/$/, ''),
                type: a.href.endsWith('/') ? 'directory' : 'file',
                path: itemPath.replace(/\/$/, '')
              };
            })
            .filter(item => !item.path.startsWith('.'));
        } catch (e) {
          console.error('Chyba při načítání struktury:', e);
          return [];
        }
      }

      static async buildTree(parentEl, basePath = '') {
        const items = await this.getDirStructure(basePath);

        items.forEach(async item => {
          if (item.name === 'index.html' || item.name === '') return;

          const li = document.createElement('li');
          li.className = `tree-item ${item.type === 'directory' ? 'tree-item--dir' : 'tree-item--file'}`;

          if (item.type === 'directory') {
            const toggle = document.createElement('div');
            toggle.className = 'tree-item__toggle';
            toggle.textContent = '▶';
            li.appendChild(toggle);

            const childrenList = document.createElement('ul');
            childrenList.className = 'tree-item__children';

            toggle.addEventListener('click', async () => {
              li.classList.toggle('tree-item--open');
              toggle.textContent = li.classList.contains('tree-item--open') ? '▼' : '▶';

              if (li.classList.contains('tree-item--open') && !childrenList.children.length) {
                await this.buildTree(childrenList, item.path + '/');
              }
            });

            li.appendChild(childrenList);
          }

          const content = document.createElement('a');
          content.className = 'tree-item__content';
          content.textContent = item.name;

          if (item.type === 'file') {
            content.href = `#${item.path.replace('.md', '')}`;
            content.addEventListener('click', () => {
              document.querySelectorAll('.tree-item__content').forEach(link => {
                link.classList.remove('active');
              });
              content.classList.add('active');
            });
          }

          li.prepend(content);
          parentEl.appendChild(li);
        });
      }

      static async loadContent(path) {
        try {
          const response = await fetch(`docs/${path}.md`);
          if (!response.ok) throw new Error('Not found');
          const md = await response.text();
          document.getElementById('content').innerHTML = marked.parse(md);
        } catch (e) {
          document.getElementById('content').innerHTML = `
            <h1>404</h1>
            <p>Dokument "${path}" nenalezen</p>
          `;
        }
      }
    }

    // Inicializace
    document.addEventListener('DOMContentLoaded', async () => {
      const tree = document.getElementById('tree');
      await LiveDocLoader.buildTree(tree);

      // Načíst obsah podle URL hash
      const initialPath = window.location.hash.slice(1) || 'README';
      LiveDocLoader.loadContent(initialPath);

      // Automaticky otevřít cestu
      const pathSegments = initialPath.split('/');
      let currentPath = '';

      for (const segment of pathSegments) {
        currentPath = currentPath ? `${currentPath}/${segment}` : segment;
        const link = document.querySelector(`a[href="#${currentPath}"]`);
        if (link) {
          let parentLi = link.closest('.tree-item');
          while (parentLi) {
            parentLi.classList.add('tree-item--open');
            const toggle = parentLi.querySelector('.tree-item__toggle');
            if (toggle) toggle.textContent = '▼';
            parentLi = parentLi.parentElement.closest('.tree-item');
          }
        }
      }
    });

    window.addEventListener('hashchange', () => {
      const path = window.location.hash.slice(1);
      LiveDocLoader.loadContent(path);
    });
  </script>
</body>
</html>
```

**Požadavky pro fungování:**

1. Server musí mít povolený directory listing pro adresář `/docs`
2. Všechny MD soubory musí být v adresáři `/docs` nebo jeho podsložkách
3. Adresáře musí končit lomítkem v directory listing výpisu

**Výhody tohoto řešení:**

- Žádný build proces - okamžité zobrazení nových souborů
- Automatická detekce změn ve struktuře
- Rekurzivní načítání podsložek při rozkliknutí
- Podpora libovolné hloubky vnoření
- Žádné závislosti kromě Marked.js

**Nastavení serveru:**

- Pro Apache přidejte do `.htaccess`:
  
  ```apache
  Options +Indexes
  IndexIgnoreReset ON
  IndexIgnore .htaccess
  ```
  
- Pro Nginx nastavte v konfiguraci:
  
  ```nginx
  location /docs/ {
  autoindex on;
  }
  ```
  

**Struktura projektu:**

```
.
├── index.html
└── docs/
    ├── README.md
    ├── guide/
    │   ├── instalace.md
    │   └── konfigurace.md
    └── api/
        ├── reference.md
        └── examples/
            └── prvni-priklad.md
```

---


Toto řešení poskytuje plně dynamickou dokumentaci, která se automaticky aktualizuje při jakékoli změně v adresáři docs, bez nutnosti ručního zásahu nebo buildovacího procesu.

Toto řešení je vhodné pro:

✅ Dokumentační systémy

✅ Knowledge base
