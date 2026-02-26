# Babylon.js GUI

## Table of Contents
- [Setup](#setup)
- [AdvancedDynamicTexture](#advanceddynamictexture)
- [Controls](#controls)
- [Containers](#containers)
- [Layout & Sizing](#layout--sizing)
- [Events](#events)
- [Performance](#performance)

## Setup

```typescript
// NPM
import { AdvancedDynamicTexture } from "@babylonjs/gui/2D/advancedDynamicTexture";
import { TextBlock } from "@babylonjs/gui/2D/controls/textBlock";
import { Button } from "@babylonjs/gui/2D/controls/button";
import { Control } from "@babylonjs/gui/2D/controls/control";
import { StackPanel } from "@babylonjs/gui/2D/controls/stackPanel";
import { Grid } from "@babylonjs/gui/2D/controls/grid";
import { Rectangle } from "@babylonjs/gui/2D/controls/rectangle";
import { Image } from "@babylonjs/gui/2D/controls/image";
import { Slider } from "@babylonjs/gui/2D/controls/sliders/slider";
import { Checkbox } from "@babylonjs/gui/2D/controls/checkbox";
import { InputText } from "@babylonjs/gui/2D/controls/inputText";
import { ScrollViewer } from "@babylonjs/gui/2D/controls/scrollViewers/scrollViewer";
```

## AdvancedDynamicTexture

```typescript
// Fullscreen overlay (most common)
const gui = AdvancedDynamicTexture.CreateFullscreenUI("UI");

// On mesh surface
const gui = AdvancedDynamicTexture.CreateForMesh(plane, 1024, 1024);

// Adaptive scaling
gui.idealWidth = 1920;  // or idealHeight
// All pixel values scale relative to this
```

## Controls

### TextBlock
```typescript
const text = new TextBlock("label", "Hello World");
text.color = "white";
text.fontSize = 24;
text.fontFamily = "Arial";
text.textWrapping = true;
text.textHorizontalAlignment = Control.HORIZONTAL_ALIGNMENT_LEFT;
text.textVerticalAlignment = Control.VERTICAL_ALIGNMENT_TOP;
gui.addControl(text);
```

### Button
```typescript
// Simple text button
const btn = Button.CreateSimpleButton("btn", "Click Me");
btn.width = "150px";
btn.height = "40px";
btn.color = "white";
btn.background = "green";
btn.cornerRadius = 5;
btn.onPointerClickObservable.add(() => {
  // handle click
});
gui.addControl(btn);

// Image button
const imgBtn = Button.CreateImageButton("imgBtn", "Label", "icon.png");

// Image only button
const iconBtn = Button.CreateImageOnlyButton("iconBtn", "icon.png");
```

### Image
```typescript
const img = new Image("img", "texture.png");
img.width = "200px";
img.height = "200px";
img.stretch = Image.STRETCH_UNIFORM; // STRETCH_NONE, STRETCH_FILL, STRETCH_UNIFORM, STRETCH_EXTEND
gui.addControl(img);
```

### Slider
```typescript
const slider = new Slider("slider");
slider.minimum = 0;
slider.maximum = 100;
slider.value = 50;
slider.width = "200px";
slider.height = "20px";
slider.color = "green";
slider.background = "gray";
slider.onValueChangedObservable.add((value) => { /* handle */ });
gui.addControl(slider);
```

### Checkbox
```typescript
const cb = new Checkbox("cb");
cb.width = "20px";
cb.height = "20px";
cb.isChecked = true;
cb.color = "green";
cb.onIsCheckedChangedObservable.add((value) => { /* handle */ });
```

### InputText
```typescript
const input = new InputText("input");
input.width = "200px";
input.height = "40px";
input.color = "white";
input.background = "#333";
input.placeholderText = "Enter text...";
input.onTextChangedObservable.add((ev) => { /* ev.text */ });
```

## Containers

### Rectangle
```typescript
const rect = new Rectangle("rect");
rect.width = "300px";
rect.height = "200px";
rect.color = "white";
rect.background = "#333";
rect.cornerRadius = 10;
rect.thickness = 2;
rect.addControl(childControl);
gui.addControl(rect);
```

### StackPanel
```typescript
const panel = new StackPanel("panel");
panel.isVertical = true;  // or false for horizontal
panel.width = "200px";
// Children need explicit height (vertical) or width (horizontal)
panel.addControl(child1);
panel.addControl(child2);
```

### Grid
```typescript
const grid = new Grid("grid");
grid.addColumnDefinition(0.5);        // 50% width
grid.addColumnDefinition(100, true);  // 100px fixed
grid.addColumnDefinition(0.5);        // 50% width
grid.addRowDefinition(0.5);           // 50% height
grid.addRowDefinition(0.5);           // 50% height

grid.addControl(control, row, col);   // place control in cell
gui.addControl(grid);
```

### ScrollViewer
```typescript
const sv = new ScrollViewer("scroll");
sv.width = "300px";
sv.height = "400px";
sv.barColor = "green";
sv.barBackground = "#333";
sv.addControl(largeContent);
gui.addControl(sv);

// Performance: freeze when content is static
sv.freezeControls = true;
```

## Layout & Sizing

```typescript
// Size: pixels or percentage
control.width = "200px";    // fixed pixels
control.width = "50%";      // percentage of parent
control.width = 0.5;        // same as "50%"

// Padding
control.paddingTop = "10px";
control.paddingLeft = "10px";

// Alignment
control.horizontalAlignment = Control.HORIZONTAL_ALIGNMENT_LEFT;   // LEFT, CENTER, RIGHT
control.verticalAlignment = Control.VERTICAL_ALIGNMENT_TOP;        // TOP, CENTER, BOTTOM

// Position offset (from alignment anchor)
control.left = "20px";
control.top = "20px";

// Z-order
control.zIndex = 10;  // higher = on top

// Visibility
control.isVisible = true;
control.isEnabled = true;
control.alpha = 0.8;
```

## Events

```typescript
// Pointer events (all controls)
control.onPointerDownObservable.add((eventData) => {});
control.onPointerUpObservable.add((eventData) => {});
control.onPointerClickObservable.add((eventData) => {});
control.onPointerEnterObservable.add((control) => {});
control.onPointerOutObservable.add((control) => {});
control.onPointerMoveObservable.add((eventData) => {});

// Block pointer events from reaching scene
control.isPointerBlocker = true;

// Focus events (InputText)
control.onFocusObservable.add(() => {});
control.onBlurObservable.add(() => {});
```

## Performance

```typescript
// Bitmap caching for complex controls
control.useBitmapCache = true;

// Disable invalidation rect optimization if rendering issues
gui.useInvalidateRectOptimization = false;

// Freeze scroll viewer content
scrollViewer.freezeControls = true;

// Disable pointer move on texture for performance
const gui = AdvancedDynamicTexture.CreateForMesh(mesh, 1024, 1024, false);
```
