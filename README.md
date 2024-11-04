# Charsm

Charsm is a port of the gorgeous [Lipgloss](https://github.com/charmbracelet/lipgloss) library from Charm CLI, part of their impressive suite of CLI tools. Definitely check out Charm’s collection of [tools](https://charm.sh/); they’re fantastic.

I’m a huge fan of CLI tools and have been building a lot of them lately. Naturally, I want my CLIs to look amazing, which is exactly what Charm CLI tools achieve. Not wanting to Go without that same polish in JavaScript, I created Charsm! For details on how I ported Lipgloss using WebAssembly, see the **porting lipgloss with wasm** section below.

If you’re looking to build beautiful TUIs, this library is for you!

![temp placeholder](https://raw.githubusercontent.com/SfundoMhlungu/Assets-for-Software-Design-Documents/refs/heads/main/charsm.png)

## Installation

Install from npm with your favorite package manager:

```bash
pnpm add charsm
```

## Getting Started

### Initialization

```js
import {initLip, Lipgloss} from "charsm"

(async function() {
    const isInit = await initLip(); // returns false if WASM fails to load, otherwise true

    if (!isInit) return; // handle failure case
})();
```

Once WASM is loaded, you can create a `Lipgloss` instance:

```js
(async function() {
    const lip = new Lipgloss();
})();
```

### Creating Styles

At its core, Charsm lets you define styles similar to CSS, which can then be applied to text.

```js
(async function() {
    // Define a style
    lip.createStyle({
        id: "primary",
        canvasColor: { color: "#7D56F4" },
        border: { type: "rounded", background: "#0056b3", sides: [true] },
        padding: [6, 8, 6, 8],
        margin: [0, 0, 8, 0],
        bold: true,
        align: 'center',
        width: 10,
        height: 12,
    });

    // Apply the style
    const result = lip.apply({ value: "🔥🦾🍕" });
    console.log(result); // Output styled result

    // Apply a specific style by ID
    const custom = lip.apply({ value: "🔥🦾🍕", id: "primary" });
    console.log(custom);
})();
```

### Style Options

Here’s an overview of the options available for creating styles:

```js
type LipglossPos = "bottom" | "top" | "left" | "right" | "center";
type BorderType = "rounded" | "block" | "thick" | "double";

interface Style {
    id: string;
    canvasColor?: { color?: string, background?: string };
    border?: { type: BorderType, foreground?: string, background?: string, sides: Array<boolean> };
    padding?: Array<number>;
    margin: Array<number>;
    bold?: boolean;
    alignV?: LipglossPos;
    alignH?: LipglossPos;   // buggy don't work
    width?: number;
    height?: number;
    maxWidth?: number;
    maxHeight?: number;
}
```
alignV works!

> **Note:** For horizontal alignment(alignH), use padding and margins.

### Padding and Margins

- One value applies to all sides: `[1]`
- Two values apply to vertical and horizontal sides: `[1, 2]`
- Four values apply to top, right, bottom, and left: `[1, 2, 3, 4]`

### Simple Example

```js
    lip.createStyle({
        id: "primary",
        canvasColor: { color: "#7D56F4" },
        border: { type: "rounded", background: "#0056b3", sides: [true] },
        padding: [6, 8, 6, 8],
        margin: [0, 2, 8, 2],
        bold: true,
        align: 'center',
        width: 10,
        height: 12,
    });;

 lip.createStyle({
    id: "secondary",
  canvasColor: {color: "#7D56F4" },
  border: { type: "rounded", background: "#0056b3", sides: [true, false] },
  padding: [6, 8, 6, 8],
   margin: [0, 0, 8, 1],
    bold: true,
    // alignH: "right",

   alignV: "bottom",
   width: 10, 
   height: 12,

  });


const a = lip.apply({ value: "Charsmmm", id: "secondary" });
const b = lip.apply({ value: "🔥🦾🍕", id: "primary" });
const c = lip.apply({ value: 'Charsmmm', id: "secondary" });
```

## Layout

Charsm currently supports horizontal and vertical layouts.

```js
const res = lip.join({ direction: "horizontal", elements: [a, b, c], position: "left" });
console.log(res);
```

> For details on `lipgloss.JoinVertical` and `lipgloss.JoinHorizontal`, refer to Charm’s lipgloss repo.

## Creating Tables

Charsm can create tables easily. Here’s an example:

```js
const rows = [
    ["Chinese", "您好", "你好"],
    ["Japanese", "こんにちは", "やあ"],
    ["Arabic", "أهلين", "أهلا"],
    ["Russian", "Здравствуйте", "Привет"],
    ["Spanish", "Hola", "¿Qué tal?"]
];

const tableData = { headers: ["LANGUAGE", "FORMAL", "INFORMAL"], rows: rows };

const t = lip.newTable({
    data: tableData,
    table: { border: "rounded", color: "99", width: 100 },
    header: { color: "212", bold: true },
    rows: { even: { color: "246" } }
});

console.log(t);
```

## Porting Lipgloss with WASM

The implementation here is a straightforward 1-to-1 port! In other words, for example `createStyle` is built up from a bunch of lipgloss functions with conditional checks. It’s verbose, kind of repetitive, and maybe even a bit annoying.

The reason for this verbosity is to avoid using `reflect` for dynamic calls to lipgloss functions, `reflect` in Go is a form of metaprogramming that's super expensive.

Here's an example of `Join`:

```go
func (l *lipWrapper) Join(this js.Value, args []js.Value) interface{} {
	direction := args[0].Get("direction").String()

	var elements []string
	e := args[0].Get("elements")
	for i := 0; i < e.Length(); i++ {
		elements = append(elements, e.Index(i).String())
	}

	if CheckTruthy(args, "pc") {
		if direction == "vertical" {
			return lipgloss.JoinVertical(lipgloss.Position(args[0].Get("pc").Int()), elements...)
		} else {
			return lipgloss.JoinHorizontal(lipgloss.Position(args[0].Get("pc").Int()), elements...)
		}
	}

	if CheckTruthy(args, "position") {
		pos := args[0].Get("position").String()
		var apos lipgloss.Position

		if pos == "bottom" {
			apos = lipgloss.Bottom
		} else if pos == "top" {
			apos = lipgloss.Top
		} else if pos == "right" {
			apos = lipgloss.Right
		} else {
			apos = lipgloss.Left
		}

		if direction == "vertical" {
			return lipgloss.JoinVertical(apos, elements...)
		} else {
			return lipgloss.JoinHorizontal(apos, elements...)
		}
	}

	return ""
}
```

That's why some features like adaptive colors aren’t implemented just yet—those will come later!

## Plan

Next up, I’m planning to port Bubble Tea for interactive components!

## Contribution

This project came up while I was building a CLI tool in JavaScript to monitor websites. I wanted it to look nice, and since I’ve been using lipgloss a lot in Go, I figured I'd port it.

Meaning, yes, the Go code is all over the place! Here’s a look at `main` for context:

```go
func main() {

	lip := &lipWrapper{}
	lip.styles = make(map[string]string)
	lip.styles2o = make(map[string]lipgloss.Style)

	// Export the `add` function to JavaScript
	// js.Global().Set("add", js.FuncOf(add))
	// js.Global().Set("greet", js.FuncOf(greet))
	// js.Global().Set("multiply", js.FuncOf(multiply))
	// js.Global().Set("processUser", js.FuncOf(processUser))
	// js.Global().Set("asyncAdd", js.FuncOf(asyncAdd))
	// js.Global().Set("lipprint", js.FuncOf(printWithGloss))
	// js.Global().Set("lipgloss", js.Func(lipgloss.NewStyle))
	js.Global().Set("createStyle", js.FuncOf(lip.createStyle))
	js.Global().Set("apply", js.FuncOf(lip.apply))
	// js.Global().Set("canvasColor", js.FuncOf(lip.canvasColor))
	// js.Global().Set("padding", js.FuncOf(lip.canvasColor))
	// js.Global().Set("render", js.FuncOf(lip.render))
	// js.Global().Set("margin", js.FuncOf(lip.margin))
	// js.Global().Set("place", js.FuncOf(lip.place))
	// js.Global().Set("size", js.FuncOf(lip.size))
	// js.Global().Set("JoinHorizontal", js.FuncOf(lip.JoinHorizontal))
	// js.Global().Set("JoinVertical", js.FuncOf(lip.JoinVertical))
	// js.Global().Set("border", js.FuncOf(lip.border))
	// js.Global().Set("width", js.FuncOf(lip.width))
	// js.Global().Set("height", js.FuncOf(lip.height))
	js.Global().Set("newTable", js.FuncOf(lip.newTable))
	// js.Global().Set("tableStyle", js.FuncOf(lip.tableStyle))
	js.Global().Set("join", js.FuncOf(lip.Join))

	// Example user input
	// input := "lipgloss.NewStyle().Foreground(lipgloss.Color(fg)).Background(lipgloss.Color(bg))"

	// // Assuming user provides these values
	// fg := "#FF0000" // red
	// bg := "#00FF00" // green

	// // style := buildStyleFromInput(input, fg, bg)

	// // Print styled text to see the result
	// styledText := style.Render("Hello, Styled World!")
	// fmt.Println(styledText)
	// // Keep the program running (WebAssembly runs until manually stopped)
	select {} // loop
}
```
yeah really bad and that's just main.

I’ve got files everywhere, so I’ll need to clean it up once I find the time then I'll post the Golang code.

## Notes on Building an Executable

To turn your Node application into an executable, make sure your build tool copies and bundles the WASM file in `charsm`’s `dist` folder.

Since it’s read with `fs` (not imported), your bundler needs to know about this file:

```javascript
const wasmPath = path.resolve(dir, './lip.wasm');
const wasmfile = fs.readFileSync(wasmPath);
```

**Disclaimer ⚠️:** This following instructions are generated by GPT, so I haven’t fully tested the bundling process yet, but I do use `pkg` to create an exe.

## Documentation: Bundling the Node Application with `charsm` WASM File

This guide covers how to bundle a Node.js application that uses the `charsm` library and its `lip.wasm` file into a standalone executable. We'll review setup for common tools like `pkg`, `nexe`, and `electron-builder`.

### Step 1: Accessing the WASM File in `charsm`

To bundle, you’ll need a dynamic reference to `lip.wasm` since its path will change in the executable.

1. **Development Path:** Typically, `node_modules/charsm/dist/lip.wasm`.
2. **Bundled Path:** Dynamically reference the WASM file at runtime.


### Step 2: Bundling with `pkg`

To include `lip.wasm`:

1. **Update `package.json`:**

   ```json
   {
     "pkg": {
       "assets": [
         "node_modules/charsm/dist/lip.wasm"
       ]
     }
   }
   ```

2. **Bundle the Application:**

   ```bash
   pkg . --assets node_modules/charsm/dist/lip.wasm
   ```

### Step 3: Bundling with `nexe`

1. **Run `nexe` with Resource Flag:**

   ```bash
   nexe -i index.js -o myApp.exe --resource node_modules/charsm/dist/lip.wasm
   ```

2. **Update Code for `process.cwd()`:**

   ```javascript
   const wasmPath = path.join(process.cwd(), 'node_modules/charsm/dist/lip.wasm');
   ```

### Step 4: Bundling with `electron-builder` (For Electron Apps)

1. **Modify `electron-builder` Configuration:**

   ```json
   {
     "files": [
       "dist/**/*",
       "node_modules/charsm/dist/lip.wasm"
     ]
   }
   ```

2. **Reference with `__dirname`:**

   ```javascript
   const wasmPath = path.join(__dirname, 'node_modules/charsm/dist/lip.wasm');
   ```

### Conclusion

Each bundling tool has a different configuration to include the `lip.wasm` file. Following these steps will ensure `charsm`’s WASM file is properly included in your executable.