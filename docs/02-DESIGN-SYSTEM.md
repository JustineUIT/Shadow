# 02 — DESIGN SYSTEM: Một nguồn token, hai platform khớp nhau

> Đọc trước khi viết dòng UI đầu tiên trên iOS hoặc web.
> Nguồn sự thật duy nhất: `contracts/tokens/tokens.json` (DTCG). Không có hex, không có số px/pt viết tay trong code UI. Lệch = CI fail (`make gen-tokens && git diff --exit-code`).

## 0. Mục tiêu và phạm vi

| Câu hỏi | Trả lời |
|---|---|
| Vì sao cần design system khi chỉ 1 người làm? | Vì có **2 platform** (SwiftUI + React) và **AI viết code**. Không có token → AI tự bịa màu/khoảng cách khác nhau ở mỗi màn, sửa tới sửa lui. |
| Bare minimum để ship là gì? | Mục §8 cột "Bare". Tóm tắt: light mode, 1 palette 5 màu semantic, 10 vai chữ, 11 component, contrast AA, Dynamic Type tới AX3, không hardcode. |
| Perfect là gì? | Cột "Perfect": dark mode, high-contrast, snapshot test, Figma sync, motion/haptics tokens, iPad. Làm **sau khi có user**, không làm trước. |
| Ai sở hữu? | Vai "Design" (bạn). Đổi token = PR vào `contracts/tokens/**` + generated files + screenshot before/after. |

---

## 1. Quyết định (options → chọn → lý do)

| # | Hạng mục | Options | CHỌN | Lý do |
|---|---|---|---|---|
| DS-D01 | Định dạng token | Viết tay 2 nơi / Figma Variables export / **DTCG JSON + Style Dictionary v4** | **DTCG JSON** | Chuẩn W3C, Style Dictionary v4 đọc trực tiếp (`usesDtcg: true`), sinh Swift + CSS |
| DS-D02 | Font | 1 font custom cả 2 platform / **System font iOS + Inter web** | **iOS: SF Pro (system). Web: Inter (latin + vietnamese)** | SF Pro là tối ưu nhất trên iOS (Dynamic Type miễn phí, hỗ trợ IPA). Inter là font gần SF nhất trên web, có Vietnamese + IPA Extensions. Đúng 2 font family cho toàn hệ |
| DS-D03 | Màu thương hiệu | Xanh dương / tím / cam / **teal đậm** | **Teal `#0F766E`** | Bình tĩnh, tập trung (sản phẩm học tập); không trùng 4 màu rating (đỏ/hổ phách/xanh lá/xanh dương); tránh gradient tím cliché |
| DS-D04 | Dark mode | Ngay từ đầu / **Perfect tier** | **Sau** | Token đã tách primitive/semantic nên thêm dark chỉ là thêm 1 file override, không refactor |
| DS-D05 | Component web | Tự viết / MUI / **shadcn/ui map sang `--ds-*`** | **shadcn/ui** | Đã dùng cho admin (D18); chỉ cần map biến CSS, không đổi cách dùng |
| DS-D06 | Component iOS | Thư viện bên thứ ba / **tự viết trong package `DesignSystem`** | **Tự viết** | SwiftUI đủ; 11 component nhỏ; kiểm soát a11y |
| DS-D07 | Đơn vị | px/pt tuyệt đối / **scale 4pt** | **4pt base** | Khớp HIG và Tailwind (`1 = 4px`) |
| DS-D08 | Kích cỡ chữ web vs iOS | Web 16 / iOS 17 khác nhau / **cùng một scale** | **Cùng scale (body 17)** | Parity tuyệt đối; 17px trên web vẫn đọc tốt |
| DS-D09 | Tên token | camelCase / **kebab-case, prefix `ds`** | `ds-color-bg-canvas` (CSS), `DS.Color.bgCanvas` (Swift) | Style Dictionary transform tự động |

---

## 2. Cấu trúc `tokens.json` (3 lớp)

```
primitive  →  semantic  →  component (Perfect, chưa làm)
color.teal.700  →  color.accent.default  →  button.primary.bg
```

**Quy tắc bất biến**: code UI **chỉ** dùng lớp *semantic*. Lớp *primitive* chỉ được tham chiếu bên trong `tokens.json`. Lint: grep `DS.Color.teal` hoặc `--ds-color-teal-` trong `ios/Packages/Features`, `web/src/features` phải = 0 kết quả.

### 2.1 Nhóm token và số lượng (Bare)

| Nhóm | Số token | Ghi chú |
|---|---|---|
| `color.primitive` | teal 50–900, zinc 50–950, red/amber/green/blue mỗi màu 3 nấc (400/600/700) | Chỉ tham chiếu nội bộ |
| `color.semantic` | 24 (bảng §3) | Code UI dùng nhóm này |
| `space` | 11 nấc | 0, 1, 2, 3, 4, 5, 6, 8, 10, 12, 16 (× 4pt) |
| `radius` | 5 | sm 6, md 10, lg 16, xl 24, full 9999 |
| `size` | 6 | touch-min 44, icon-sm 16, icon-md 20, icon-lg 24, card-max-w 480, content-max-w 640 |
| `typography` | 10 vai | bảng §4 |
| `motion` | 3 duration + 2 easing | fast 120 ms, base 200 ms, slow 320 ms; standard `cubic-bezier(0.2, 0, 0, 1)`, emphasized `cubic-bezier(0.3, 0, 0, 1)` |
| `elevation` | 2 | sm `0 1px 2px rgba(0,0,0,.06)`, md `0 4px 12px rgba(0,0,0,.08)` |
| `breakpoint` (web only) | 4 | sm 640, md 768, lg 1024, xl 1280 |
| `z` (web only) | 5 | base 0, dropdown 10, sticky 20, overlay 30, toast 40 |

### 2.2 Ví dụ cấu trúc (trích, không phải file đầy đủ)

```json
{
  "color": {
    "teal": { "700": { "$type": "color", "$value": "#0F766E" } },
    "zinc": { "50": { "$type": "color", "$value": "#FAFAFA" } },
    "bg": {
      "canvas":  { "$type": "color", "$value": "{color.zinc.50}" },
      "surface": { "$type": "color", "$value": "#FFFFFF" }
    },
    "accent": {
      "default": { "$type": "color", "$value": "{color.teal.700}" },
      "fg":      { "$type": "color", "$value": "#FFFFFF" }
    }
  },
  "space": { "4": { "$type": "dimension", "$value": "16px" } },
  "typography": {
    "body": {
      "$type": "typography",
      "$value": { "fontFamily": "{font.family.body}", "fontSize": "17px", "fontWeight": 400, "lineHeight": "22px" },
      "$extensions": { "ios.textStyle": "body" }
    }
  }
}
```

`$extensions.ios.textStyle` là cầu nối để Swift dùng `Font.TextStyle` (Dynamic Type) thay vì size cố định.

---

## 3. Bảng màu semantic (light) — đã đo contrast

Contrast tính theo WCAG 2.x, **đo bằng script** (xem checklist DS-05). Yêu cầu: chữ ≥ 4.5:1, chữ lớn (≥ 24px hoặc ≥ 19px bold) và UI component ≥ 3:1.

| Token | Hex | Dùng cho | Contrast đo được | Đạt |
|---|---|---|---|---|
| `color.bg.canvas` | `#FAFAFA` | nền màn hình | — | |
| `color.bg.surface` | `#FFFFFF` | card, sheet, input | — | |
| `color.bg.subtle` | `#F4F4F5` | hàng xen kẽ, disabled bg | — | |
| `color.bg.accent-subtle` | `#CCFBF1` | badge/chip nhạt | — | |
| `color.fg.primary` | `#18181B` | chữ chính | 16.97 trên canvas, 17.72 trên surface | AAA |
| `color.fg.secondary` | `#52525B` | chữ phụ | 7.73 trên surface | AAA |
| `color.fg.muted` | `#71717A` | caption, placeholder | 4.83 surface / 4.63 canvas | AA |
| `color.fg.on-accent` | `#FFFFFF` | chữ trên nút primary | 5.47 trên accent.default | AA |
| `color.fg.on-accent-subtle` | `#134E4A` | chữ trên accent-subtle | 8.41 | AAA |
| `color.accent.default` | `#0F766E` | nút primary, link, focus | 5.47 (làm chữ trên surface) | AA |
| `color.accent.hover` | `#115E59` | hover/pressed | 7.58 với chữ trắng | AAA |
| `color.border.default` | `#E4E4E7` | viền card, divider | 1.27 (trang trí, không cần đạt) | n/a |
| `color.border.strong` | `#8E8E96` | viền input, viền có ý nghĩa | 3.25 trên surface | AA (UI ≥ 3:1) |
| `color.focus.ring` | `#0F766E` | focus visible | 5.47 | AA |
| `color.status.success` | `#15803D` | text/icon | 5.02 | AA |
| `color.status.warning` | `#B45309` | text/icon | 5.02 | AA |
| `color.status.danger` | `#B91C1C` | text/icon, nút destructive | 6.47 | AA |
| `color.status.info` | `#1D4ED8` | text/icon | 6.70 | AA |
| `color.rating.again` | `#DC2626` | nền nút Again (chữ trắng) | 4.83 | AA |
| `color.rating.hard` | `#B45309` | nền nút Hard (chữ trắng) | 5.02 | AA |
| `color.rating.good` | `#15803D` | nền nút Good (chữ trắng) | 5.02 | AA |
| `color.rating.easy` | `#2563EB` | nền nút Easy (chữ trắng) | 5.17 | AA |
| `color.state.new` | `#2563EB` | badge trạng thái thẻ mới | 5.17 | AA |
| `color.state.learning` | `#B45309` | badge learning/relearning | 5.02 | AA |
| `color.state.review` | `#15803D` | badge review | 5.02 | AA |

**Đếm màu**: 1 brand (teal) + 3 neutral (zinc trắng/xám/đen) + 1 nhóm accent trạng thái (đỏ/hổ phách/xanh lá/xanh dương dùng chung cho status và rating) = đúng giới hạn 5 "màu" của design guideline.

**Quy tắc màu rating**: màu **không bao giờ là kênh thông tin duy nhất**. Nút rating luôn có nhãn chữ ("Again / Hard / Good / Easy" hoặc tiếng Việt) + interval preview. Người mù màu đỏ-lục vẫn phân biệt được bằng vị trí (luôn thứ tự 1→4 trái sang phải) và nhãn.

### 3.1 Dark mode (Perfect — giá trị đã chuẩn bị, chưa bật)

| Token | Dark hex | Contrast đo được |
|---|---|---|
| `bg.canvas` | `#09090B` | — |
| `bg.surface` | `#18181B` | — |
| `fg.primary` | `#FAFAFA` | 16.97 trên surface |
| `fg.secondary` | `#A1A1AA` | 6.91 |
| `fg.muted` | `#8F8F98` | 5.53 (không dùng `#71717A`: chỉ 3.67, fail) |
| `accent.default` | `#2DD4BF` | 9.52 làm chữ trên surface |
| `fg.on-accent` | `#042F2E` | 7.77 trên accent |
| `rating.again` / fg | `#F87171` / `#450A0A` | 5.84 |
| `rating.hard` / fg | `#F59E0B` / `#1C1917` | 8.14 |
| `rating.good` / fg | `#22C55E` / `#052E16` | 6.54 |
| `rating.easy` / fg | `#60A5FA` / `#172554` | 5.78 |

File: `contracts/tokens/tokens.dark.json` chỉ override lớp semantic. CSS ra `[data-theme="dark"]` + `@media (prefers-color-scheme: dark)`; Swift ra `Color(light:dark:)` qua `UIColor { traits in ... }`.

---

## 4. Typography scale (10 vai, cùng số trên 2 platform)

| Vai | iOS `Font.TextStyle` | Size (pt/px) | Weight | Line-height | Dùng cho |
|---|---|---|---|---|---|
| `display` | `.largeTitle` | 34 | 700 | 41 | Số streak, hero onboarding |
| `title1` | `.title` | 28 | 700 | 34 | Tiêu đề màn hình |
| `title2` | `.title2` | 22 | 600 | 28 | Tiêu đề section |
| `title3` | `.title3` | 20 | 600 | 25 | Lemma trên FlashCard |
| `headline` | `.headline` | 17 | 600 | 22 | Tiêu đề hàng list, tên deck |
| `body` | `.body` | 17 | 400 | 22 | Định nghĩa, nội dung |
| `callout` | `.callout` | 16 | 400 | 21 | Ví dụ câu |
| `subhead` | `.subheadline` | 15 | 400 | 20 | Meta thứ cấp |
| `footnote` | `.footnote` | 13 | 400 | 18 | Chú thích, attribution |
| `caption` | `.caption` | 12 | 400 | 16 | Nhãn badge, timestamp |

Quy tắc:
- iOS **luôn** dùng `Font.ds(.body)` → bên trong là `.system(.body, design: .default)` để Dynamic Type hoạt động. Không bao giờ `Font.system(size: 17)`.
- Web: `font-size` bằng `rem` tính từ token (`17px → 1.0625rem`), `line-height` unitless từ token. Không dùng `text-[17px]`.
- IPA (`/ˌɪntəˈnæʃənəl/`): dùng vai `callout`, không dùng monospace. Kiểm tra render không có ô vuông (tofu) trên cả 2 platform (DS-08).
- Không có vai nào < 12px. Body luôn line-height ≥ 1.3.
- Web `<html lang="vi">` hoặc `en` đúng locale để hyphenation/quotes đúng.

---

## 5. Catalogue component (Bare: 8 core + 3 product)

Ký hiệu: **M** = prop bắt buộc, **O** = tuỳ chọn. "Output" = event/callback/binding component phát ra. Mọi component phải có: Preview/Story ở 3 trạng thái tối thiểu; a11y label; không hardcode màu/space.

### 5.1 Core

| Component | Input M | Input O | Output | States | A11y MUST | Token chính |
|---|---|---|---|---|---|---|
| `DSButton` | `label`, `action` | `variant: primary\|secondary\|ghost\|destructive` (default primary), `size: md\|lg`, `icon`, `isLoading`, `isDisabled`, `fullWidth` | tap | default, pressed, disabled, loading | role button, label = text; loading → `aria-busy`/`accessibilityValue("Đang xử lý")`; min 44×44 | `accent.*`, `radius.md`, `space.3/4`, `typography.headline` |
| `DSTextField` | `label`, `value` (binding) | `placeholder`, `helper`, `error`, `keyboard: email\|number\|text`, `maxLength` (lấy từ contract), `autofocus` | value change, submit | default, focused, error, disabled | label liên kết input (`accessibilityLabel` / `<label for>`); error → `aria-describedby` + `aria-invalid`; announce error | `border.strong`, `focus.ring`, `status.danger`, `radius.md` |
| `DSSurface` | `content` | `padding: space token`, `elevated: bool`, `bordered: bool` | — | — | container thuần, không có role | `bg.surface`, `border.default`, `radius.lg`, `elevation.sm` |
| `DSListRow` | `title` | `subtitle`, `leading` (icon/badge), `trailing` (chevron/value), `action` | tap | default, pressed, disabled | 1 accessibility element gộp title+subtitle+value; hint "Double tap để mở" nếu có action | `typography.headline/subhead`, `space.4`, `size.touch-min` |
| `DSBadge` | `text` | `tone: neutral\|accent\|new\|learning\|review\|success\|warning\|danger` | — | — | text đọc được; không chỉ dùng màu | `state.*`, `status.*`, `radius.full`, `typography.caption` |
| `DSProgress` | `value 0..1` | `kind: bar\|ring`, `label`, `showPercent` | — | — | `accessibilityValue("60%")` / `role=progressbar aria-valuenow` | `accent.default`, `bg.subtle`, `radius.full` |
| `DSToast` | `message`, `kind: info\|success\|warning\|danger` | `action {label, handler}`, `duration` (default motion.slow×10) | dismiss, action | visible, dismissing | `role=status` (web) / `UIAccessibility.post(.announcement)`; không tự ẩn nếu có action và VoiceOver bật | `status.*`, `bg.surface`, `elevation.md`, `z.toast` |
| `DSEmptyState` | `title` | `message`, `illustration`, `primaryAction`, `secondaryAction` | action | — | heading level đúng; nút có label | `typography.title2/body`, `space.6/8`, `content-max-w` |

### 5.2 Product

| Component | Input M | Input O | Output | States | A11y MUST |
|---|---|---|---|---|---|
| `FlashCard` | `front: string`, `back: string`, `isFlipped` (binding) | `ipa`, `example`, `audioURL`, `state: CardState.state` (badge), `onPlayAudio` | flip (tap/space), playAudio | front, back, flipping, audio-playing | Trước lật: label = front, hint "Double tap để xem nghĩa"; sau lật: label = back (+ example); nút audio riêng label "Phát âm"; `reduceMotion` → crossfade thay flip 3D; Dynamic Type AX3 vẫn không cắt chữ (scroll trong card) |
| `RatingBar` | `intervals: {again, hard, good, easy}: string` (đã format "1 phút", "3 ngày"), `onRate(1..4)` | `isEnabled`, `layout: row\|grid` (auto grid khi AX size) | rate(rating) | enabled, disabled | 4 nút, label "Again, ôn lại sau 1 phút"; thứ tự cố định; màu không là kênh duy nhất; phím tắt web 1–4 |
| `DeckCard` | `title`, `cardCount` | `progress {new, learning, review, dueToday}`, `cefr`, `coverURL`, `visibility`, `onOpen` | open | default, pressed | 1 element gộp: "IELTS Core, 500 thẻ, 12 thẻ đến hạn hôm nay"; ảnh cover `decorative` |

Component **Perfect** (làm sau): `DSSheet`, `DSSegmentedControl`, `StreakCalendar`, `StatTile`, `PaywallPlanCard`, `AudioWaveButton`, `SkeletonRow`.

---

## 6. Ánh xạ platform

### 6.1 iOS — package `DesignSystem`

```
ios/Packages/DesignSystem/
├── Package.swift                 # không phụ thuộc Features/Core; chỉ SwiftUI
├── Sources/DesignSystem/
│   ├── Generated/Tokens.swift    # style-dictionary sinh, KHÔNG sửa tay
│   ├── Tokens+SwiftUI.swift      # Color(token:), Font.ds(_:), CGFloat.space(_:)
│   ├── Components/{DSButton,DSTextField,...}.swift
│   ├── Product/{FlashCard,RatingBar,DeckCard}.swift
│   └── Previews/Catalog.swift    # PreviewProvider duyệt toàn bộ component × 3 Dynamic Type
└── Tests/DesignSystemTests/      # snapshot (Perfect), decode tokens (Bare)
```

Quy ước sử dụng:
- `Color.ds(.bgCanvas)`, `Font.ds(.body)`, `.padding(.ds(.space4))`, `.cornerRadius(.ds(.radiusMd))`.
- Mọi `View` public của package có `#Preview` ở size `.large`, `.accessibility3`, và dark (Perfect).
- SwiftLint custom rule `no_hardcoded_color`: regex `Color\((red|\.sRGB|hex|"#)` và `Color\.(red|blue|green|gray|black|white)\b` → error trong `Features/**` và `DesignSystem/Components/**`.
- SwiftLint custom rule `no_fixed_font_size`: regex `\.system\(size:` → error.

### 6.2 Web — `web/src/design-system/`

```
web/src/design-system/
├── tokens.css        # sinh từ style-dictionary: :root { --ds-color-bg-canvas: #FAFAFA; ... }
├── theme.css         # @theme inline map --ds-* → Tailwind + shadcn vars (viết tay, ổn định)
└── index.ts          # re-export shadcn components đã map
```

`theme.css` (viết tay 1 lần, không đổi khi token đổi):

```css
@import "tailwindcss";
@import "./tokens.css";

@theme inline {
  --color-background: var(--ds-color-bg-canvas);
  --color-foreground: var(--ds-color-fg-primary);
  --color-card: var(--ds-color-bg-surface);
  --color-card-foreground: var(--ds-color-fg-primary);
  --color-primary: var(--ds-color-accent-default);
  --color-primary-foreground: var(--ds-color-fg-on-accent);
  --color-secondary: var(--ds-color-bg-subtle);
  --color-secondary-foreground: var(--ds-color-fg-primary);
  --color-muted: var(--ds-color-bg-subtle);
  --color-muted-foreground: var(--ds-color-fg-muted);
  --color-accent: var(--ds-color-bg-accent-subtle);
  --color-accent-foreground: var(--ds-color-fg-on-accent-subtle);
  --color-destructive: var(--ds-color-status-danger);
  --color-border: var(--ds-color-border-default);
  --color-input: var(--ds-color-border-strong);
  --color-ring: var(--ds-color-focus-ring);
  --radius: var(--ds-radius-md);
  --font-sans: var(--ds-font-family-body), ui-sans-serif, system-ui;
  --text-body: var(--ds-typography-body-font-size);
  /* ... các vai chữ còn lại */
}
```

Lint web: stylelint `color-no-hex` + ESLint rule cấm `className` chứa `#[0-9a-f]{3,6}` hoặc `text-[..px]` (regex trong `eslint-plugin-tailwindcss` `no-arbitrary-value` bật cho `color`, `fontSize`, `spacing`).

### 6.3 Style Dictionary config (`contracts/tokens/style-dictionary.config.mjs`)

| Mục | Giá trị |
|---|---|
| `source` | `["contracts/tokens/tokens.json"]` (+ `tokens.dark.json` cho platform dark, Perfect) |
| `usesDtcg` | `true` |
| Platform `css` | `transformGroup: "css"`, `prefix: "ds"`, `buildPath: "web/src/design-system/"`, file `tokens.css` format `css/variables`, `options.outputReferences: true`, `selector: ":root"` |
| Platform `swift` | `transformGroup: "ios-swift-separate"`, `buildPath: "ios/Packages/DesignSystem/Sources/DesignSystem/Generated/"`, file `Tokens.swift` format `ios-swift/enum.swift`, `className: "DSTokens"`, `accessControl: "public"` |
| Typography | custom format nhỏ `swift/typography` sinh `enum DSTextRole { case body ... var textStyle: Font.TextStyle }` từ `$extensions.ios.textStyle` |
| Lệnh | `pnpm -w style-dictionary build -c contracts/tokens/style-dictionary.config.mjs` (alias `make gen-tokens`) |
| CI | `contract.yml` job `tokens-drift`: build rồi `git diff --exit-code -- web/src/design-system/tokens.css ios/Packages/DesignSystem/Sources/DesignSystem/Generated` |

---

## 7. Layout, motion, a11y — quy tắc dùng chung

| Chủ đề | Quy tắc |
|---|---|
| Grid | Mobile-first. Nội dung đọc dài giới hạn `size.content-max-w` (640). FlashCard giới hạn `size.card-max-w` (480), căn giữa trên màn rộng |
| Spacing | Chỉ dùng token `space`. Giữa các section: `space.8`; trong card: `space.4`; giữa label và input: `space.2` |
| Touch | Mọi target ≥ 44×44 (`size.touch-min`). RatingBar cao ≥ 56 |
| Motion | Flip card `motion.slow` easing emphasized; toast in/out `motion.base`; press feedback `motion.fast`. `reduceMotion` → mọi transform thành opacity |
| Haptics (iOS) | Rating → `.impact(.light)`; hoàn thành session → `.notification(.success)`; lỗi → `.notification(.error)`. Không haptic khi VoiceOver đang đọc |
| Empty/Loading/Error | Mọi màn có dữ liệu bất đồng bộ phải có 3 trạng thái này bằng `DSEmptyState` / skeleton / `DSToast` — không màn trắng |
| Icon | iOS: SF Symbols. Web: `lucide-react`. Size chỉ 16/20/24. Icon-only button phải có label |
| Ảnh | Cover deck tỉ lệ 4:3, `cdn.<domain>/covers/<id>.webp` + AVIF; luôn có `alt` hoặc đánh dấu decorative |
| Text | `text-balance` cho tiêu đề, `text-pretty` cho đoạn (web). iOS: `.multilineTextAlignment(.leading)` mặc định, không justify |
| Dynamic Type | Test ở `.xSmall`, `.large`, `.accessibility3`. Không cắt chữ; layout đổi từ row → column khi `dynamicTypeSize.isAccessibilitySize` |
| Localization | Chuỗi UI ở `Localizable.xcstrings` (iOS) và `messages/{vi,en}.json` (web). Số/ngày format theo locale (`Intl`, `FormatStyle`) |

---

## 8. Bare Minimum vs Perfect

| Hạng mục | Bare (gate ship) | Perfect (sau) |
|---|---|---|
| Token | Light; 24 màu semantic; space/radius/size/typography/motion/elevation | Dark; high-contrast; component-layer tokens; haptics tokens |
| Sinh code | `tokens.css`, `Tokens.swift`, drift CI | Figma Variables sync (Tokens Studio) → PR tự động |
| Component | 11 (§5) với preview 3 trạng thái | +7 Perfect; Storybook web; catalog app iOS riêng |
| A11y | AA contrast đo bằng script; VoiceOver label; Dynamic Type AX3; reduceMotion | AAA cho body; Voice Control names; Switch Control test; Bold Text/Increase Contrast |
| Test | Token decode test; lint no-hardcode | Snapshot (swift-snapshot-testing, Playwright `toHaveScreenshot`) 3 size × 2 theme |
| Tài liệu | File này + preview | Docs site (Storybook) có usage do/don't |
| Layout | iPhone portrait | iPad size classes, landscape, web app học |

---

## 9. Checklist Design System (ID `02-DS-nn`)

`[M]` = bạn tự re-check. Bằng chứng = file:dòng hoặc output lệnh.

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| DS-01 | `tokens.json` hợp lệ DTCG, đủ nhóm §2.1 | Style Dictionary build 0 lỗi; số token đúng bảng | + `tokens.dark.json` | `make gen-tokens` exit 0; `jq '[..|.["$value"]?|select(.!=null)]|length' tokens.json` ≥ 80 | |
| DS-02 | Generated files khớp | `git diff --exit-code` sau build = 0 | | CI job `tokens-drift` xanh | |
| DS-03 | Không hardcode màu iOS | SwiftLint `no_hardcoded_color` 0 vi phạm | | `swiftlint lint --strict` output | |
| DS-04 | Không hardcode màu/size web | stylelint + eslint 0 vi phạm; `grep -rnE "#[0-9a-fA-F]{6}" web/src --include=*.tsx` = 0 | | output lệnh | |
| DS-05 | Contrast | Mọi cặp trong §3 ≥ 4.5 (chữ) / ≥ 3 (UI), đo bằng `scripts/contrast-check.mjs` đọc tokens.json | Dark cũng đạt | `node scripts/contrast-check.mjs` in bảng, 0 FAIL → `docs/evidence/design/<date>-contrast.txt` | |
| DS-06 | Typography | 10 vai; iOS dùng TextStyle; web dùng rem | | grep `\.system\(size:` = 0; grep `text-\[` = 0 | |
| DS-07 | Dynamic Type AX3 | 5 màn chính không cắt chữ, không overlap | xSmall → AX5 | Xcode Preview + Accessibility Inspector; ảnh chụp vào evidence | [M] |
| DS-08 | IPA render | Không tofu trên iOS + web (Safari/Chrome) với 20 IPA mẫu | | Màn debug `/design` (web) + preview iOS; screenshot | [M] |
| DS-09 | 11 component có preview 3 trạng thái | Catalog compile; mỗi component ≥ 3 preview | Snapshot test | `xcodebuild build -scheme DesignSystem`; web route `/design` (chỉ build dev/preview, noindex) | |
| DS-10 | A11y label component | Mọi control có label; icon-only có label | | Accessibility Inspector audit 0 issue; axe 0 serious trên `/design` | |
| DS-11 | Reduce Motion | Flip → crossfade; toast không slide | | Bật Reduce Motion, quay video 10 s | [M] |
| DS-12 | shadcn map đầy đủ | Mọi biến shadcn dùng trong `components/ui/*` có trong `theme.css` | | script grep `var(--` trong `components/ui` ⊆ danh sách `theme.css` | |
| DS-13 | Touch target | ≥ 44×44 mọi control | | Accessibility Inspector "Hit region"; Playwright đo `boundingBox` ≥ 44 | |
| DS-14 | Trạng thái empty/loading/error | Mọi màn async có 3 trạng thái | Skeleton chuẩn | Review từng màn theo danh sách màn trong PRD | [M] |

### Prompt cho AI (copy nguyên văn)

```
Bạn là reviewer Design System. Đầu vào: docs/02-DESIGN-SYSTEM.md, contracts/tokens/tokens.json,
thư mục ios/Packages/DesignSystem, web/src/design-system, và các file UI tôi đính kèm.
Kiểm tra từng mục DS-01 → DS-14. Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (đường dẫn:file:dòng hoặc lệnh + output rút gọn) | việc cần sửa (nếu FAIL)
Không được PASS nếu không có bằng chứng. Mục [M] chỉ được ghi "CẦN MANUAL", không tự PASS.
Cuối cùng liệt kê mọi hex/px hardcode bạn tìm được kèm đường dẫn.
```

### Re-check thủ công (bạn làm, 20 phút)

1. Mở app trên máy thật, Settings → Accessibility → Display & Text Size → Larger Text kéo tối đa. Đi qua 5 màn chính. Không được cắt chữ.
2. Bật VoiceOver, học 3 thẻ mắt nhắm. Phải biết được mình đang ở nút nào, interval bao lâu.
3. Bật Reduce Motion, lật thẻ. Phải là crossfade.
4. Mở `/design` trên web bằng bàn phím (Tab/Shift+Tab/Enter/Esc), không dùng chuột. Focus ring luôn thấy.
5. So 1 màn iOS và 1 màn web cùng nội dung: màu, khoảng cách, cỡ chữ phải giống nhau bằng mắt.

---

## 10. File output kỳ vọng

| File | Ai tạo | Khi nào |
|---|---|---|
| `contracts/tokens/tokens.json` | Design | P0-06 |
| `contracts/tokens/style-dictionary.config.mjs` | Design | P0-06 |
| `contracts/tokens/tokens.dark.json` | Design | Perfect |
| `scripts/contrast-check.mjs` | Design | P0-06 |
| `web/src/design-system/tokens.css` (generated), `theme.css` | gen / Web | P0-06, WEB-01 |
| `ios/Packages/DesignSystem/**` (Generated/Tokens.swift + 11 component + Catalog preview) | gen / iOS | IOS-02 |
| `web/app/design/page.tsx` (catalog, noindex, chỉ non-prod) | Web | WEB-01 |
| `docs/evidence/design/<date>-contrast.txt`, `<date>-dynamic-type/*.png` | QA | Cuối Phase 2/3 |
