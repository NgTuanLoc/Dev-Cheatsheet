---
tags: [react, advanced, accessibility]
aliases: [i18n, Translation, ICU MessageFormat]
level: Advanced
---

# Internationalization

> **One-liner**: i18n in React means three concerns — **translating strings**, **formatting locale-aware values** (numbers, dates, plurals, currency), and **layout** (RTL); the dominant libraries are **react-i18next**, **react-intl** (formatjs), and increasingly **lingui**.

---

## Quick Reference

| Concern | Tool / API |
|---------|-----------|
| Translation lookup | `react-i18next`, `react-intl`, `lingui`, `next-intl` |
| Plural / select / ordinal | **ICU MessageFormat** |
| Number / currency formatting | `Intl.NumberFormat` (built into JS) |
| Date / time formatting | `Intl.DateTimeFormat`, or `date-fns` / `Day.js` |
| Relative time | `Intl.RelativeTimeFormat` |
| List formatting | `Intl.ListFormat` |
| RTL layout | `dir="rtl"` on `<html>`, CSS logical properties |
| Locale detection | `Accept-Language` header, IP, user setting |
| Translation extraction | `i18next-parser`, `formatjs/cli`, lingui's CLI |

---

## Core Concept

i18n looks easy ("just swap strings") and is famously hard once plurals, gender, currency, and right-to-left languages enter the picture.

The pieces:
- **Translation lookup**: a key (`"signup.welcome"`) plus current locale → text. Bundled per-locale or fetched async.
- **ICU MessageFormat**: a tiny DSL inside translated strings for plural, select (gender, status), number formatting. `"{count, plural, one {# item} other {# items}}"`.
- **Locale-aware formatting**: never hand-format dates/numbers/currency. Use the browser's `Intl.*` APIs (or libraries that wrap them) — they handle decimal separators, currency symbols, calendar systems.
- **RTL**: Arabic, Hebrew, Persian, Urdu read right-to-left. Use CSS *logical* properties (`margin-inline-start` instead of `margin-left`) so layouts flip automatically.

For Next.js / Remix, framework-aware libraries (`next-intl`, Remix-i18next) integrate with routing for locale-prefixed URLs.

---

## Syntax & API

### react-i18next setup

```bash
npm install i18next react-i18next
```

```ts
// i18n.ts
import i18n from "i18next";
import { initReactI18next } from "react-i18next";

i18n.use(initReactI18next).init({
  lng: "en",
  fallbackLng: "en",
  interpolation: { escapeValue: false },
  resources: {
    en: { common: {
      welcome: "Hello, {{name}}",
      "items.count": "{{count}} item",
      "items.count_other": "{{count}} items",
    }},
    es: { common: {
      welcome: "Hola, {{name}}",
      "items.count": "{{count}} artículo",
      "items.count_other": "{{count}} artículos",
    }},
  },
});

export default i18n;
```

```tsx
import { useTranslation } from "react-i18next";

function Greeting({ name, count }: { name: string; count: number }) {
  const { t, i18n } = useTranslation("common");

  return (
    <>
      <h1>{t("welcome", { name })}</h1>
      <p>{t("items.count", { count })}</p>          {/* auto-pluralizes by count */}

      <button onClick={() => i18n.changeLanguage(i18n.language === "en" ? "es" : "en")}>
        Switch language
      </button>
    </>
  );
}
```

### react-intl (formatjs) — ICU MessageFormat

```bash
npm install react-intl
```

```tsx
import { IntlProvider, FormattedMessage, FormattedNumber, FormattedDate } from "react-intl";

const messages = {
  en: { greeting: "Hello, {name}!", items: "{count, plural, one {# item} other {# items}}" },
  es: { greeting: "¡Hola, {name}!", items: "{count, plural, one {# artículo} other {# artículos}}" },
};

function App({ locale }: { locale: "en" | "es" }) {
  return (
    <IntlProvider locale={locale} messages={messages[locale]}>
      <FormattedMessage id="greeting" values={{ name: "Ana" }} />
      <FormattedMessage id="items"    values={{ count: 5 }} />
      <FormattedNumber value={1234567.89} style="currency" currency="EUR" />
      <FormattedDate value={new Date()} year="numeric" month="long" day="numeric" />
    </IntlProvider>
  );
}
```

### `Intl.*` directly (no library)

```ts
new Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" }).format(1234.56);
// "1.234,56 €"

new Intl.DateTimeFormat("ja-JP").format(new Date());
// "2026/5/5"

new Intl.RelativeTimeFormat("en", { numeric: "auto" }).format(-1, "day");
// "yesterday"

new Intl.ListFormat("en", { type: "conjunction" }).format(["A", "B", "C"]);
// "A, B, and C"
```

### Next.js App Router with `next-intl`

```bash
npm install next-intl
```

```tsx
// middleware.ts
import createMiddleware from "next-intl/middleware";
export default createMiddleware({ locales: ["en", "es"], defaultLocale: "en" });

// app/[locale]/layout.tsx
import { NextIntlClientProvider } from "next-intl";
import { getMessages } from "next-intl/server";

export default async function Layout({ children, params: { locale } }) {
  const messages = await getMessages();
  return (
    <html lang={locale} dir={locale === "ar" ? "rtl" : "ltr"}>
      <body>
        <NextIntlClientProvider messages={messages}>{children}</NextIntlClientProvider>
      </body>
    </html>
  );
}

// In a server or client component
import { useTranslations } from "next-intl";
const t = useTranslations("nav");
<a>{t("home")}</a>
```

### RTL with CSS logical properties

```css
/* Use logical (work in both LTR and RTL) */
.card {
  padding-inline: 1rem;       /* both left + right */
  margin-inline-start: 0.5rem; /* "start" flips to right in RTL */
  text-align: start;          /* not "left" */
  border-inline-start: 1px solid #ccc;
}
```

```tsx
<html dir={locale === "ar" ? "rtl" : "ltr"}>
```

---

## Common Patterns

```tsx
// Pattern: locale-aware money formatter as a hook
function useMoney(locale: string, currency: string) {
  return useMemo(
    () => new Intl.NumberFormat(locale, { style: "currency", currency }),
    [locale, currency],
  );
}
const fmt = useMoney("en-US", "USD");
<p>{fmt.format(1234.56)}</p>

// Pattern: ICU select for gender
"{gender, select, female {She liked it} male {He liked it} other {They liked it}}"
```

---

## Gotchas & Tips

- **Don't concatenate translated strings.** "Hello " + name + "!" breaks word order in many languages. Use placeholders.
- **Don't pluralize with ternary.** `count === 1 ? "item" : "items"` is wrong in Slavic, Arabic, etc. (multiple plural forms). Use ICU plural or library helpers.
- **Currency requires a code** (`USD`, `EUR`), not a symbol. `Intl.NumberFormat` handles symbols and decimal places per currency.
- **Dates in JS are timezone-fragile.** Always store UTC ISO strings; format with the user's locale + timezone.
- **RTL is more than text direction.** Mirror icons (back arrows, chart axes), test layouts, watch logical-property fallbacks.
- **Translation files must be loaded.** Either bundle them (small apps) or lazy-load by locale with namespaces (large apps).
- **Translators are humans.** Provide context and screenshots. Tools like Crowdin, Lokalise, Phrase plug into i18next/formatjs.
- **Performance**: large translation bundles bloat first paint. Code-split per locale; only load the active language.
- **SSR**: detect locale from `Accept-Language` or URL. Mismatch between server and client → hydration error.
- **Pseudo-localize during dev** (`Wéllćòmé!`) to find untranslated strings and overflowing layouts.

---

## See Also

- [[20 - Accessibility]]
- [[14 - Design Systems]]
- [[08 - Next.js App Router]]
