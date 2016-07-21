A **style** is a collection of properties that specify the look and format for a View or window.
```xml
<TextView
    android:layout_width="fill_parent"
    android:layout_height="wrap_content"
    android:textColor="#00FF00"
    android:typeface="monospace"
    android:text="@string/hello" />
```
and turn it into this:
```xml
<TextView
    style="@style/CodeFont"
    android:text="@string/hello" />
```
A **theme** is a style applied to an **entire Activity or application**, rather than an **individual View** (as in the example above). 

### Inheritance

```xml
<style name="GreenText" parent="@android:style/TextAppearance">
        <item name="android:textColor">#00FF00</item>
</style>
```

If you want to inherit from styles that you've defined yourself, you do not have to use the **parent attribute**.

Instead, just prefix the name of the style you want to inherit to the name of your new style, **separated by a period.**

For example, to create a new style that inherits the CodeFont style defined above, but make the color red, you can author the new style like this:

```xml
<style name="CodeFont.Red">
        <item name="android:textColor">#FF0000</item>
</style>
```

Notice that there is no parent attribute in the <style> tag, but because the name attribute begins with the CodeFont style name (which is a style that you have created), this style inherits all style properties from that style. This style then overrides the android:textColor property to make the text red. You can reference this new style as @style/CodeFont.Red.

You can continue inheriting like this as **many times** as you'd like, by chaining names with periods. For example, you can extend CodeFont.Red to be bigger, with:

```xml
<style name="CodeFont.Red.Big">
        <item name="android:textSize">30sp</item>
</style>
```

This inherits from both CodeFont and CodeFont.Red styles, then adds the android:textSize property.