# PreSonus Shopify Theme - Header Hover Navigation Guide

This guide documents the implementation of the **Dynamic Multi-Menu Hover Navigation** for the Shopify Horizon Header. 

All changes have been made exclusively within the Liquid template files without modifying any asset JavaScript files (like `header.js` or `header-menu.js`), maintaining a beginner-friendly code structure.

---

## 1. Key Problem Solved: Hydration & Shadow DOM

We resolved two critical challenges during development:
1. **Shopify Section Hydration:** Because the header element uses lazy hydration, direct event listeners attached on `DOMContentLoaded` are destroyed and lost once hydration completes. We solved this by using **event delegation** on the `document` level.
2. **Shadow DOM Events:** The header's link lists use an `<overflow-list>` component which renders links in a Shadow DOM. When hover events bubble across shadow boundaries, the browser retargets `event.target` to the host. We resolved this by using **`event.composedPath()`** to traverse shadow roots and successfully find `data-menu-slug` attributes.

---

## 2. Modifications Made in Code

### File 1: [`blocks/_header-menu.liquid`](file:///Users/goharkhalid/Documents/PreSonus/blocks/_header-menu.liquid)
We added `data-menu-slug` attributes to the desktop list items (`<li>`) and links (`<a>`) in the menu loop:

```liquid
  {% for link in block_settings.menu.links %}
    <li class="menu-list__list-item" data-menu-slug="{{ link.handle }}">
      ...
      <a href="{{ link.url }}" data-menu-slug="{{ link.handle }}" class="menu-list__link">
      ...
    </li>
  {% endfor %}
```
*(Example: If the link title is "Shop", it generates `<a data-menu-slug="shop">`).*

---

### File 2: [`sections/header.liquid`](file:///Users/goharkhalid/Documents/PreSonus/sections/header.liquid) (Schema settings)
We added 3 text input settings allowing merchants to enter the hover parent slug in the Shopify Customizer:

```json
  {
    "type": "text",
    "id": "hover_slug_1",
    "label": "Hover Slug 1",
    "default": "shop"
  },
  {
    "type": "text",
    "id": "hover_slug_2",
    "label": "Hover Slug 2",
    "default": "support"
  },
  {
    "type": "text",
    "id": "hover_slug_3",
    "label": "Hover Slug 3",
    "default": "learn"
  }
```

---

### File 3: [`sections/header.liquid`](file:///Users/goharkhalid/Documents/PreSonus/sections/header.liquid) (Menu Level Wrappers)
We attached the `data-menu-slug` attributes to the secondary levels:

```liquid
  <div class="level_one_menu menu_levels active_menu" data-menu-slug="{{ section.settings.hover_slug_1 }}">
    {{ menu_one }}
  </div>
  <div class="level_two_menu menu_levels inactive_menu" data-menu-slug="{{ section.settings.hover_slug_2 }}">
    {{ menu_two }}
  </div>
  <div class="level_three_menu menu_levels inactive_menu" data-menu-slug="{{ section.settings.hover_slug_3 }}">
    {{ menu_three }}
  </div>
```

---

### File 4: [`sections/header.liquid`](file:///Users/goharkhalid/Documents/PreSonus/sections/header.liquid) (Inline JavaScript Controller)
We added this script to the bottom of the section to manage hover events dynamically:

```html
<script>
  document.addEventListener('DOMContentLoaded', function () {
    const blocks = document.querySelectorAll('.menu_levels');
    const headerComponent = document.querySelector('header-component');

    function deactivateAll() {
      blocks.forEach(function(block) {
        block.classList.add('inactive_menu');
        block.classList.remove('active_menu');
      });
      document.querySelectorAll('[data-menu-slug]').forEach(function(trigger) {
        trigger.classList.remove('active');
      });
    }

    // Event delegation with composedPath support for Shadow DOM boundary crossing
    document.addEventListener('mouseover', function(event) {
      let trigger = null;
      const path = event.composedPath();
      for (let i = 0; i < path.length; i++) {
        const node = path[i];
        if (node.getAttribute && node.getAttribute('data-menu-slug')) {
          trigger = node;
          break;
        }
      }

      if (trigger) {
        const slug = trigger.getAttribute('data-menu-slug');
        if (slug) {
          const targetBlock = document.querySelector('.menu_levels[data-menu-slug="' + slug + '"]');
          if (targetBlock) {
            deactivateAll();
            targetBlock.classList.add('active_menu');
            targetBlock.classList.remove('inactive_menu');
            
            // Add active class to all matching triggers
            document.querySelectorAll('[data-menu-slug="' + slug + '"]').forEach(function(el) {
              el.classList.add('active');
            });

            // Close active submenus inside other blocks
            document.querySelectorAll('header-menu').forEach(function(menu) {
              if (menu !== targetBlock.querySelector('header-menu')) {
                menu.setAttribute('aria-expanded', 'false');
                const activeSub = menu.querySelector('[data-active]');
                if (activeSub) {
                  delete activeSub.dataset.active;
                  activeSub.setAttribute('inert', '');
                }
              }
            });

            // Reset submenu heights
            if (headerComponent) {
              headerComponent.style.setProperty('--submenu-height', '0px');
              headerComponent.style.setProperty('--full-open-header-height', '0px');
            }
          }
        }
      }
    });

    // Event delegation for clicks (touch devices)
    document.addEventListener('click', function(event) {
      let trigger = null;
      const path = event.composedPath();
      for (let i = 0; i < path.length; i++) {
        const node = path[i];
        if (node.getAttribute && node.getAttribute('data-menu-slug')) {
          trigger = node;
          break;
        }
      }

      if (trigger) {
        const slug = trigger.getAttribute('data-menu-slug');
        if (slug) {
          const targetBlock = document.querySelector('.menu_levels[data-menu-slug="' + slug + '"]');
          if (targetBlock) {
            // Prevent default page navigation if it has a matching submenu block
            event.preventDefault();
            
            deactivateAll();
            targetBlock.classList.add('active_menu');
            targetBlock.classList.remove('inactive_menu');
            
            document.querySelectorAll('[data-menu-slug="' + slug + '"]').forEach(function(el) {
              el.classList.add('active');
            });
          }
        }
      }
    });
  });
</script>
```

---

## 3. Dashboard Guide for Merchants

To customize which menu shows up under which tab:
1. Go to **Shopify Admin** -> **Online Store** -> **Themes** -> **Customize**.
2. Open the **Header** section settings.
3. Under the **Dynamic Hover Menus** settings, locate **Hover Slug 1**, **Hover Slug 2**, and **Hover Slug 3**.
4. Type in the lowercase handle (slug) of the parent link from your Main Menu:
   * E.g., if the parent menu item is "Shop", the handle is `shop`.
   * E.g., if the parent menu item is "Support", the handle is `support`.
5. Save the settings. Hovering over those menu items will toggle their respective menus and keep the active one visible.
