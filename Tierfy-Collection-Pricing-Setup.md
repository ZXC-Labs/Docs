# Tierfy: Customer Tag-Based Pricing Setup Guide

Display discounted wholesale/VIP prices directly on collection pages and automatically hide the tier table on product pages for single-tier flat discounts.

---

## 📋 Overview of Changes
1. **Create 1 Snippet:** `snippets/tierfy-collection-pricing.liquid` (handles the logic)
2. **Add 1 Line to Theme:** In `layout/theme.liquid` (`{% render 'tierfy-collection-pricing' %}`)
3. **Wrap Price in Collection:** In `snippets/price.liquid` (adds the product data wrapper)

---

## 📍 Step 1: Create the Snippet File

1. In your Shopify Admin, go to **Online Store → Themes**.
2. Click the **three dots (`...`)** next to your active theme and select **Edit code**.
3. Under the **Snippets** folder on the left, click **Add a new snippet**.
4. Name the snippet: `tierfy-collection-pricing` (it will create `tierfy-collection-pricing.liquid`).
5. Paste the following code into the file and click **Save**:

```liquid
<!-- Tierfy Collection Pricing & PDP Control -->
<script>
(function() {
    const currentCustomerTags = "{% if customer %}{{ customer.tags | join: ',' | downcase }}{% endif %}";
    const currencySymbol = "{{ cart.currency.symbol }}";

    // 1. Product Detail Page (PDP): Hide table if it only has 1 tier for Min Qty = 1
    function checkAndHidePdpTable() {
        const table = document.getElementById('tiered-pricing-table-wrapper');
        if (!table) return;

        const rows = table.querySelectorAll('.tp-row, .tpm-row, .tpn-row, .tpp-row, .tpw-row, .tps-card, tbody tr');
        if (rows.length === 1) {
            const text = rows[0].textContent || '';
            const isMinQty1 = /\b1\s*(\+|unit|piece|item|ea|qty)/i.test(text) || text.includes('1+');
            if (isMinQty1) {
                table.style.setProperty('display', 'none', 'important');
                const skeleton = document.getElementById('tiered-pricing-skeleton');
                if (skeleton) skeleton.style.setProperty('display', 'none', 'important');
                return;
            }
        }
        if (rows.length > 1 && table.style.display === 'none') {
            table.style.removeProperty('display');
        }
    }

    // 2. Collection Page Discount Display
    function loadTieredPrices() {
        const priceElements = Array.from(document.querySelectorAll('.tierfy-collection-price:not(.tierfy-processed)'));
        if (priceElements.length === 0) return;
        priceElements.forEach(el => el.classList.add('tierfy-processed'));

        let allProductIds = [], allTags = [], allCollectionIds = [], allSkus = [];
        priceElements.forEach(el => {
            if (el.dataset.productId) allProductIds.push(el.dataset.productId);
            if (el.dataset.tags) allTags = allTags.concat(el.dataset.tags.split(',').map(s => s.trim()));
            if (el.dataset.collectionIds) allCollectionIds = allCollectionIds.concat(el.dataset.collectionIds.split(',').map(s => s.trim()));
            if (el.dataset.skus) allSkus = allSkus.concat(el.dataset.skus.split(',').map(s => s.trim()));
        });

        const uniqueProductIds = [...new Set(allProductIds)].filter(Boolean).join(',');
        const uniqueTags = [...new Set(allTags)].filter(Boolean).join(',');
        const uniqueCollectionIds = [...new Set(allCollectionIds)].filter(Boolean).join(',');
        const uniqueSkus = [...new Set(allSkus)].filter(Boolean).join(',');

        const url = `/apps/tiered-pricing?product_id=${encodeURIComponent(uniqueProductIds)}` +
                    `&tags=${encodeURIComponent(uniqueTags)}` +
                    `&collection_ids=${encodeURIComponent(uniqueCollectionIds)}` +
                    `&skus=${encodeURIComponent(uniqueSkus)}` +
                    `&customer_tags=${encodeURIComponent(currentCustomerTags)}`;

        fetch(url)
          .then(res => res.json())
          .then(data => {
              if (!data.rules || data.rules.length === 0) return;

              priceElements.forEach(el => {
                  const pData = {
                      id: el.dataset.productId,
                      price: parseFloat(el.dataset.productPrice || 0),
                      tags: (el.dataset.tags || '').split(',').map(s => s.trim()),
                      collectionIds: (el.dataset.collectionIds || '').split(',').map(s => s.trim()),
                      skus: (el.dataset.skus || '').split(',').map(s => s.trim())
                  };

                  const matchingRules = data.rules.filter(rule => {
                      if (rule.customerSelection !== 'TAGS') return false;
                      let rVals = [];
                      try { rVals = JSON.parse(rule.targetValues); } catch(e) { rVals = rule.targetValues.split(','); }
                      rVals = rVals.map(v => String(v).toLowerCase().trim());
                      if (rule.targetType === 'PRODUCT') return rVals.some(v => v.includes(String(pData.id)));
                      if (rule.targetType === 'COLLECTION') return rVals.some(v => pData.collectionIds.includes(v.split('/').pop()));
                      if (rule.targetType === 'TAG') return rVals.some(v => pData.tags.includes(v));
                      if (rule.targetType === 'SKU') return rVals.some(v => pData.skus.includes(v));
                      return false;
                  });

                  if (matchingRules.length === 0) return;

                  let allTiers = [];
                  matchingRules.forEach(r => {
                      if (r.tiers) {
                          allTiers = allTiers.concat(r.tiers.map(t => {
                              let savings = 0;
                              if (t.discountType === 'PERCENTAGE') {
                                  savings = Math.floor((pData.price * 100 * t.discountValue) / 100) / 100;
                              } else {
                                  savings = parseFloat(t.discountValue);
                              }
                              return { ...t, _savings: savings };
                          }));
                      }
                  });

                  if (allTiers.length === 0) return;

                  const tierQty1 = allTiers.find(t => parseInt(t.minQuantity, 10) === 1);
                  const maxSavingsTier = allTiers.reduce((prev, current) => (prev._savings > current._savings) ? prev : current);

                  let newHtml = '';
                  const originalPriceStr = pData.price.toFixed(2);
                  const originalPriceHtml = `<span style="text-decoration: line-through; opacity: 0.6; margin-right: 6px;">${currencySymbol}${originalPriceStr}</span>`;

                  if (tierQty1 && allTiers.length === 1) {
                      const newPrice = (pData.price - tierQty1._savings).toFixed(2);
                      newHtml = `${originalPriceHtml} <span style="color: #e32c2b; font-weight: bold;">${currencySymbol}${newPrice}</span>`;
                  } else if (allTiers.length > 1) {
                      const minPrice = (pData.price - maxSavingsTier._savings).toFixed(2);
                      newHtml = `<span style="color: #e32c2b; font-weight: bold;">As low as ${currencySymbol}${minPrice}</span>`;
                  } else if (!tierQty1) {
                      const newPrice = (pData.price - maxSavingsTier._savings).toFixed(2);
                      newHtml = `<span style="font-weight: bold; color: #e32c2b;">Buy ${maxSavingsTier.minQuantity}+ for ${currencySymbol}${newPrice}/ea</span>`;
                  }

                  const ogPriceEl = el.querySelector('.original-theme-price');
                  if (ogPriceEl) ogPriceEl.style.display = 'none';

                  el.insertAdjacentHTML('beforeend', `<div class="tierfy-smart-price">${newHtml}</div>`);
              });
          })
          .catch(err => console.error("Tierfy Error:", err));
    }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', function() {
            checkAndHidePdpTable();
            loadTieredPrices();
        });
    } else {
        checkAndHidePdpTable();
        loadTieredPrices();
    }

    // Monitor for both PDP table injection and AJAX collection pagination/filters
    const observer = new MutationObserver((mutations) => {
        checkAndHidePdpTable();

        let shouldLoad = false;
        mutations.forEach((mutation) => {
            if (mutation.addedNodes.length) {
                mutation.addedNodes.forEach(node => {
                    if (node.nodeType === 1) {
                        if (node.classList && node.classList.contains('tierfy-collection-price') && !node.classList.contains('tierfy-processed')) {
                            shouldLoad = true;
                        } else if (node.querySelector && node.querySelector('.tierfy-collection-price:not(.tierfy-processed)')) {
                            shouldLoad = true;
                        }
                    }
                });
            }
        });
        if (shouldLoad) {
            clearTimeout(window.__tpCollectionTimeout);
            window.__tpCollectionTimeout = setTimeout(loadTieredPrices, 50);
        }
    });

    observer.observe(document.body, { childList: true, subtree: true });
})();
</script>
```

---

## 📍 Step 2: Render in `theme.liquid`

1. In the search bar on the left, search for `theme.liquid` under the **Layout** folder.
2. Scroll to the very bottom and locate the closing `</body>` tag.
3. Paste this **single line** right before `</body>` and click **Save**:

```liquid
{% render 'tierfy-collection-pricing' %}
```

---

## 📍 Step 3: Wrap the Price in `price.liquid`

1. In the search bar on the left, search for `price.liquid` (or `card-product.liquid`) under **Snippets**.
2. Find the main price container and **wrap it** with `<div class="tierfy-collection-price">`:

```liquid
{%- assign tierfy_product = product_resource | default: product | default: card_product -%}
<div class="tierfy-collection-price"
     data-product-id="{{ tierfy_product.id | default: product.id | default: card_product.id }}"
     data-product-price="{{ tierfy_product.price | default: product.price | default: card_product.price | divided_by: 100.0 }}"
     data-tags="{{ tierfy_product.tags | default: product.tags | default: card_product.tags | join: ',' | downcase | escape }}"
     data-collection-ids="{{ tierfy_product.collections | default: product.collections | default: card_product.collections | map: 'id' | join: ',' }}"
     data-skus="{{ tierfy_product.variants | default: product.variants | default: card_product.variants | map: 'sku' | join: ',' | downcase | escape }}">
  
  <div class="original-theme-price">
    <!-- YOUR EXISTING PRICE CONTAINER / LIQUID CODE HERE -->
  </div>

</div>
```
3. Click **Save**.

---

## 📍 Step 4: Test & Verify

1. Log into your Shopify storefront with a customer account tagged with your customer rule (e.g., `VIP` or `Wholesale`).
2. Open any **Collection page** to verify that discounted prices appear directly in the grid.
3. Open a product with a 1-tier flat discount on the **Product Page** to verify that the redundant table is hidden.

🎉 **All done!**
