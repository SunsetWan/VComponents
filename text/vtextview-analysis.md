# VTextView 实现分析与 iOS 14 兼容建议

本文整理两个问题：

1. `VTextView` 为什么底层仍然使用 SwiftUI 的 `TextField` 实现，而不是 `TextEditor` 或 UIKit 的 `UITextView`。
2. 如果要写一个类似 `VTextView` 的组件，但最低支持 iOS 14，应该如何设计。

## 1. 为什么 `VTextView` 用的是 SwiftUI `TextField`

`VTextView` 这个名字容易让人联想到 UIKit 里的 `UITextView`，但当前仓库里的 `VTextView` 并不是一个完整的长文本编辑器封装。它更像是 VComponents 输入组件体系里的「多行表单输入框」。

核心代码在 `Sources/VComponents/Presentation/Components/Inputs/TextView/VTextView.swift`：

```swift
TextField(
    text: $text,
    prompt: placeholder.map {
        Text($0)
            .textConfiguration(appearance.placeholderTextConfiguration, state: internalState)
    },
    axis: .vertical,
    label: EmptyView.init
)
```

也就是说，`VTextView` 的主体确实是 SwiftUI `TextField`，只是使用了 `axis: .vertical`，让输入内容可以沿垂直方向增长或滚动。这是 iOS 16 之后 SwiftUI 提供的多行 `TextField` 能力。

### 1.1 组件语义更接近「多行 TextField」

从公开 API 和 appearance 配置可以看出，`VTextView` 关注的是表单输入框常见能力：

- placeholder。
- focus 状态。
- enabled / focused / disabled 三态颜色。
- background 和 border。
- corner radius。
- content margins。
- keyboard type。
- text content type。
- autocorrection。
- autocapitalization。
- submit label。
- text configuration 和 placeholder text configuration。

这些能力都和 `TextField` 的使用模型高度一致。

相比之下，真正的长文本编辑器通常还会关心：

- 独立滚动行为。
- 大段文本编辑性能。
- 光标和选区控制。
- 富文本或 attributed string。
- 文本查找、撤销栈、编辑菜单等复杂编辑体验。

当前 `VTextView` 没有暴露这些编辑器级别的能力。因此它的定位不是「类似 Notes 的文本编辑器」，而是「可多行的输入框」。

### 1.2 与 `VTextField` 保持同一套组件模型

`VTextField` 在 `Sources/VComponents/Presentation/Components/Inputs/TextField/VTextField.swift` 中也基于 SwiftUI text input 控件构建。它在输入框外层叠加了：

- search image。
- clear button。
- secure visibility button。
- fixed height。
- background。
- border。
- focus state。

`VTextView` 和 `VTextField` 共享了很多设计理念：

- 都由 SwiftUI 原生输入控件承载实际输入行为。
- 都通过 appearance model 控制视觉样式。
- 都使用 `@FocusState` 记录 focus。
- 都把背景点击映射为 focus。
- 都使用 `.textFieldStyle(.plain)`，再由组件自己控制外观。
- 都复用 `textConfiguration`、keyboard、content type、autocorrection、submit label 等修饰符。

所以 `VTextView` 使用 `TextField(axis: .vertical)` 的一个重要收益是：它可以和 `VTextField` 保持非常接近的实现结构和行为模型。

如果改成 `TextEditor`，很多 `TextField` 专属或更自然的行为就不能直接复用，例如 prompt、text field style、submit 行为，以及部分表单输入语义。

### 1.3 `TextField(axis: .vertical)` 解决了多行输入和自适应高度问题

仓库历史里有一条提交：

```text
d6b4e7246 Fix issue with resizing in VTextView
```

这次修改把 `axis: .vertical` 打开，并把组件可用性提高到支持该 API 的系统版本。这个历史说明：作者并不是不小心用了 `TextField`，而是明确依赖 SwiftUI 多行 `TextField` 来解决 resizing / 多行增长问题。

现在 package 最低平台是：

```swift
platforms: [
    .iOS(.v17),
    .macOS(.v14),
    .tvOS(.v17),
    .watchOS(.v10),
    .visionOS(.v1)
]
```

因此当前代码可以放心使用 iOS 16+ 之后的多行 `TextField` API，不需要兼容 iOS 14/15。

### 1.4 为什么没有用 `TextEditor`

SwiftUI 的 `TextEditor` 官方定位是 long-form text editor，也就是更适合编辑长文本。它当然可以实现多行输入，但对当前 `VTextView` 的需求来说不一定更合适。

`TextEditor` 的主要问题是：

- 没有和 `TextField` 一样直接的 `prompt` initializer。
- placeholder 需要自己叠加。
- 背景、内边距、滚动区域样式在早期 iOS 上不好控制。
- focus、submit、keyboard 行为和 `TextField` 不完全一致。
- 很难和 `VTextField` 保持同一套轻量输入框抽象。

如果组件目标只是「多行备注框、评论框、表单描述输入框」，`TextField(axis: .vertical)` 更贴近需求。

### 1.5 这个选择的边界

当前实现适合：

- 表单里的备注、描述、评论。
- 短到中等长度的多行输入。
- 需要 placeholder、focus border、keyboard type、submit label 的统一输入体验。
- 希望和 `VTextField` 外观体系一致的场景。

当前实现不适合：

- 大段长文编辑。
- 富文本编辑。
- 类似 Notes、Markdown editor、code editor 的场景。
- 需要复杂选区、滚动、编辑菜单、查找替换的场景。

如果未来要支持这些更重的编辑器能力，应该考虑：

- `TextEditor`。
- UIKit `UITextView` wrapper。
- 更专业的自定义编辑器。

## 2. iOS 14 下如何写类似 `VTextView` 的组件

如果最低支持 iOS 14，就不能照搬当前 `VTextView` 的实现。原因是 `TextField(..., axis: .vertical, ...)` 不是 iOS 14 可用能力。

iOS 14 下有三个方向：

1. 使用 SwiftUI `TextEditor`。
2. 使用 `UIViewRepresentable` 包装 UIKit `UITextView`。
3. 尝试用 iOS 14 的 `TextField` 模拟多行。

第三种不建议。iOS 14 的 `TextField` 没有垂直 axis，多行、换行、动态高度、输入法、光标和选区行为都很难做对。

### 2.1 方案 A：`TextEditor` + 自定义外观

这是最简单的 iOS 14 兼容方案。

适合：

- 备注输入。
- 评论输入。
- 表单描述。
- 不需要特别精细的 UIKit 行为控制。
- 想保持纯 SwiftUI。

基本结构：

```swift
public struct LegacyTextView: View {
    private let placeholder: String?
    @Binding private var text: String

    public init(
        placeholder: String? = nil,
        text: Binding<String>
    ) {
        self.placeholder = placeholder
        self._text = text
    }

    public var body: some View {
        ZStack(alignment: .topLeading) {
            if text.isEmpty, let placeholder {
                Text(placeholder)
                    .foregroundColor(.secondary)
                    .padding(.top, 8)
                    .padding(.leading, 5)
            }

            TextEditor(text: $text)
                .padding(0)
        }
        .padding(EdgeInsets(top: 8, leading: 10, bottom: 8, trailing: 10))
        .frame(minHeight: 50, alignment: .top)
        .background(Color(.secondarySystemBackground))
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}
```

这个版本能工作，但 iOS 14/15 的 `TextEditor` 有几个需要注意的点：

- placeholder 需要自己画。
- `TextEditor` 默认背景可能挡住自定义背景。
- padding 和 text container inset 不像 `UITextView` 那样可精确控制。
- focus 状态在 iOS 14 没有 `@FocusState`，需要自己通过 UIKit 或回调模拟。

如果只做轻量组件，这个方案足够。

### 2.2 方案 B：`UITextView` wrapper

如果要做组件库里的稳定基础组件，更推荐用 UIKit `UITextView` 包一层 SwiftUI。

适合：

- 需要 iOS 14。
- 需要可靠 placeholder。
- 需要 begin editing / end editing 状态。
- 需要精确控制内边距。
- 需要 keyboard type、return key、content type。
- 需要动态高度。
- 需要更接近系统 `UITextView` 的行为。

基本结构：

```swift
import SwiftUI
import UIKit

public struct LegacyTextView: UIViewRepresentable {
    @Binding private var text: String

    private let placeholder: String?
    private let minHeight: CGFloat
    private let keyboardType: UIKeyboardType
    private let textContentType: UITextContentType?
    private let autocorrectionType: UITextAutocorrectionType
    private let autocapitalizationType: UITextAutocapitalizationType
    private let returnKeyType: UIReturnKeyType
    private let onEditingChanged: ((Bool) -> Void)?

    public init(
        placeholder: String? = nil,
        text: Binding<String>,
        minHeight: CGFloat = 50,
        keyboardType: UIKeyboardType = .default,
        textContentType: UITextContentType? = nil,
        autocorrectionType: UITextAutocorrectionType = .default,
        autocapitalizationType: UITextAutocapitalizationType = .sentences,
        returnKeyType: UIReturnKeyType = .default,
        onEditingChanged: ((Bool) -> Void)? = nil
    ) {
        self.placeholder = placeholder
        self._text = text
        self.minHeight = minHeight
        self.keyboardType = keyboardType
        self.textContentType = textContentType
        self.autocorrectionType = autocorrectionType
        self.autocapitalizationType = autocapitalizationType
        self.returnKeyType = returnKeyType
        self.onEditingChanged = onEditingChanged
    }

    public func makeUIView(context: Context) -> UITextView {
        let textView = UITextView()

        textView.delegate = context.coordinator
        textView.backgroundColor = .clear
        textView.font = .preferredFont(forTextStyle: .body)
        textView.adjustsFontForContentSizeCategory = true
        textView.textContainerInset = UIEdgeInsets(top: 0, left: 0, bottom: 0, right: 0)
        textView.textContainer.lineFragmentPadding = 0
        textView.keyboardType = keyboardType
        textView.textContentType = textContentType
        textView.autocorrectionType = autocorrectionType
        textView.autocapitalizationType = autocapitalizationType
        textView.returnKeyType = returnKeyType

        return textView
    }

    public func updateUIView(_ textView: UITextView, context: Context) {
        if textView.text != text {
            textView.text = text
        }

        textView.keyboardType = keyboardType
        textView.textContentType = textContentType
        textView.autocorrectionType = autocorrectionType
        textView.autocapitalizationType = autocapitalizationType
        textView.returnKeyType = returnKeyType
    }

    public func makeCoordinator() -> Coordinator {
        Coordinator(
            text: $text,
            onEditingChanged: onEditingChanged
        )
    }

    public final class Coordinator: NSObject, UITextViewDelegate {
        private let text: Binding<String>
        private let onEditingChanged: ((Bool) -> Void)?

        init(
            text: Binding<String>,
            onEditingChanged: ((Bool) -> Void)?
        ) {
            self.text = text
            self.onEditingChanged = onEditingChanged
        }

        public func textViewDidChange(_ textView: UITextView) {
            text.wrappedValue = textView.text
        }

        public func textViewDidBeginEditing(_ textView: UITextView) {
            onEditingChanged?(true)
        }

        public func textViewDidEndEditing(_ textView: UITextView) {
            onEditingChanged?(false)
        }
    }
}
```

实际做成类似 `VTextView` 的组件时，不建议让 `UIViewRepresentable` 自己负责所有外观。更好的拆分是：

- `UITextViewRepresentable`：只负责输入行为。
- `LegacyTextView` SwiftUI wrapper：负责 background、border、corner radius、placeholder、padding、disabled/focused 状态。

这样外观逻辑仍然保留在 SwiftUI 层，UIKit wrapper 只处理 iOS 14 下 SwiftUI 缺失的输入能力。

### 2.3 iOS 14 下 focus 怎么处理

当前 `VTextView` 用的是：

```swift
@FocusState private var isFocused: Bool
```

但 `@FocusState` 是后续系统才可用，iOS 14 不能依赖它。

iOS 14 下可以改成：

```swift
@State private var isFocused: Bool = false
```

然后从 `UITextViewDelegate` 回调同步：

```swift
textViewDidBeginEditing -> isFocused = true
textViewDidEndEditing -> isFocused = false
```

如果还想支持外部主动 focus，可以在 wrapper 里传一个 binding：

```swift
@Binding var isFocused: Bool
```

在 `updateUIView` 中根据 binding 调用：

```swift
if isFocused, !textView.isFirstResponder {
    textView.becomeFirstResponder()
}

if !isFocused, textView.isFirstResponder {
    textView.resignFirstResponder()
}
```

这比 iOS 15+ 的 `@FocusState` 麻烦，但在 iOS 14 上更可靠。

### 2.4 推荐的最终 API 形状

如果目标是写一个类似 `VTextView` 的组件，同时最低支持 iOS 14，建议 API 不要暴露底层实现细节。

可以设计成：

```swift
public struct VLegacyTextView: View {
    public init(
        appearance: VLegacyTextViewAppearance = .init(),
        placeholder: String? = nil,
        text: Binding<String>
    )
}
```

appearance 可以包含：

```swift
public struct VLegacyTextViewAppearance {
    public var minimumHeight: CGFloat = 50
    public var contentMargins: EdgeInsets = .init(top: 15, leading: 15, bottom: 15, trailing: 15)
    public var cornerRadius: CGFloat = 12
    public var backgroundColor: Color = Color(.secondarySystemBackground)
    public var focusedBackgroundColor: Color = Color(.tertiarySystemBackground)
    public var disabledBackgroundColor: Color = Color(.systemGray6)
    public var borderWidth: CGFloat = 0
    public var borderColor: Color = .clear
    public var focusedBorderColor: Color = .clear
    public var textFont: Font = .body
    public var placeholderFont: Font = .body
    public var textColor: Color = .primary
    public var placeholderColor: Color = .secondary
    public var keyboardType: UIKeyboardType = .default
    public var textContentType: UITextContentType?
    public var autocorrectionType: UITextAutocorrectionType = .default
    public var autocapitalizationType: UITextAutocapitalizationType = .sentences
    public var returnKeyType: UIReturnKeyType = .default
}
```

如果你希望尽量贴近现有 `VTextViewAppearance`，可以保留：

- `minimumHeight`
- `contentMargins`
- `cornerRadius`
- `backgroundColors`
- `borderWidth`
- `borderColors`
- `textConfiguration`
- `placeholderTextConfiguration`
- `keyboardType`
- `textContentType`
- `isAutocorrectionEnabled`
- `autocapitalization`
- `submitButton`

不过在 iOS 14 + UIKit wrapper 里，`SubmitLabel` 不能直接一比一映射，需要转换成 `UIReturnKeyType`。

### 2.5 我的建议

如果只是项目内部使用，且需求简单：

```text
TextEditor + placeholder overlay
```

这样代码少，维护成本低。

如果是组件库，或者你希望行为稳定：

```text
SwiftUI 外壳 + UITextViewRepresentable 内核
```

这是更稳妥的 iOS 14 方案。它不像当前 `VTextView` 那样完全依赖 SwiftUI `TextField`，但可以保留非常接近的外部 API。将来最低系统提高到 iOS 16+ 后，也可以在内部切换到 `TextField(axis: .vertical)`，而不影响调用方。

## 3. 总结

当前仓库里的 `VTextView` 基于 `TextField(axis: .vertical)`，主要是因为它的组件目标是「多行输入框」，而不是「完整文本编辑器」。这种实现能最大程度复用 `TextField` 的 placeholder、focus、keyboard、submit 和 text field style 行为，也能和 `VTextField` 保持一致的组件模型。

如果最低支持 iOS 14，就不能使用这个方案。最实际的选择是：

- 简单场景：用 `TextEditor` 加自定义 placeholder 和外观。
- 稳定组件库场景：用 `UITextView` wrapper，再用 SwiftUI 外壳实现 VComponents 风格的 appearance、focus 和边框状态。

不要尝试用 iOS 14 的 `TextField` 硬模拟多行输入。这个方向会在换行、动态高度、光标、输入法和可访问性上产生大量边界问题。
