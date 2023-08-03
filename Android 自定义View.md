# Android 自定义View

## View 属性

### 系统属性

Android 系统定义的属性，称为系统属性，它们可以在 sdk\platforms\android-xx\data\res\values\attrs.xml 文件中找到。一般以 declare-styleable 为一个组合，以 attr 作为一个属性配置，name 是属性名，format 是属性类型。

```xml
<declare-styleable name="View">
    <attr name="id" format="reference" />
    <attr name="background" format="reference|color" />
    <attr name="padding" format="dimension" />
     ...
    <attr name="focusable" format="boolean" />
     ...
</declare-styleable>

<declare-styleable name="TextView">
    <attr name="text" format="string" localization="suggested" />
    <attr name="hint" format="string" />
    <attr name="textColor" />
    <attr name="textColorHighlight" />
    <attr name="textColorHint" />
     ...
</declare-styleable>

<declare-styleable name="ViewGroup_Layout">
    <attr name="layout_width" format="dimension">
        <enum name="fill_parent" value="-1" />
        <enum name="match_parent" value="-1" />
        <enum name="wrap_content" value="-2" />
    </attr>
    <attr name="layout_height" format="dimension">
        <enum name="fill_parent" value="-1" />
        <enum name="match_parent" value="-1" />
        <enum name="wrap_content" value="-2" />
    </attr>
</declare-styleable>

<declare-styleable name="LinearLayout_Layout">
    <attr name="layout_width" />
    <attr name="layout_height" />
    <attr name="layout_weight" format="float" />
    <attr name="layout_gravity" />
</declare-styleable>

<declare-styleable name="RelativeLayout_Layout">
    <attr name="layout_centerInParent" format="boolean" />
    <attr name="layout_centerHorizontal" format="boolean" />
    <attr name="layout_centerVertical" format="boolean" />
     ...
</declare-styleable>
```

### 自定义属性

当使用 declare-styleable 创建属性组后，其中的属性可以使用已有的属性，也可以自定义。

使用已有的属性，只需要在自定义属性中声明，并加上命名空间。

```xml
<resources>
    <declare-styleable name="MyTextView">
        <attr name=“android:text"/>
    </declare-styleable>
</resources>
```

自定义属性，需要设置属性名和属性类型

```xml
<resources>
    <declare-styleable name="MyTextView">
        <attr name=“text" format="string" />
    </declare-styleable>
</resources>
```

属性类型有以下几种：

- reference

- color

- boolean

- dimension

- float

- integer

- string

- fraction

- emun

  ```xml
  <declare-styleable name="名称"> 
  	<attr name="orientation"> 
  		<enum name="horizontal" value="0" />
  		<enum name="vertical" value="1" />
  	</attr>
  </declare-styleable>
  ```

- flag

  ```xml
  <declare-styleable name="名称"> 
  	<attr name="gravity"> 
  		<flag name="top" value="0x30" /> 
  		<flag name="bottom" value="0x50" />
  		<flag name="left" value="0x03" /> 
  		<flag name="right" value="0x05" /> 
  		<flag name="center_vertical" value="0x10" />
  	</attr>
  </declare-styleable>
  ```

### 命名空间

在布局文件中使用使用属性时，会在前面加上「android:」，这个就是引入的命名空间「xmlns:android="http://schemas.android.com/apk/res/android” 」表示到 Android 系统中查找该属性的来源。

如果自定义属性，那就需要去应用程序包中查找，所以需要引入应用包的命名空间。引入命名空间可以设置自动查找或者设置为应用包名

- xmlns:tallfish="http://schemas.android.com/apk/res-auto”
- xmlns:tallfish="http://schemas.android.com/apk/com.example.tallfish”

## 解析属性

在 View 的构造函数中，需要解析布局文件中设置的属性，过程中用到了 Attributeset 和 TypedArray。

Attributeset 内部就是一个 xml 解析器，将布局文件中的该空间的所有属性解析出来，并以 key-value 的形式维护起来。直接从 Attributeset 中通过 getAttributeName()/getAttributeValue() 获取的数据，如果是资源ID格式，会有@前缀，此时数据是资源文件的ID，通过它再获取真正的数据。

在 attrs.xml 中写的 declare-styleable 会在编译期间，生成 Style 类，将自定义属性分组并记录属性的索引值。

```java
public static final class styleable
          public static final int[] MyTextView = {
            0x0101014f, 0x7f010038, 0x7f010039
        };
        public static final int MyTextView_android_text = 0;
        public static final int MyTextView_mTextColor = 1;
        public static final int MyTextView_mTextSize = 2;
｝
```

TypedArray 是 Context.obtainStyledAttributes() 获取的就是仅包括 styleable 数组的属性值

```java
public MyTextView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.MyTextView);
    String mText = ta.getString(R.styleable.MyTextView_android_text);
    int mTextColor = ta.getColor(R.styleable.MyTextView_mTextColor, Color.BLACK);
    int mTextSize = ta.getDimensionPixelSize(R.styleable.MyTextView_mTextSize, 100);
    float width = ta.getDimension(R.styleable.MyTextView_android_layout_width, 0.0f);
    float hight = ta.getDimension(R.styleable.MyTextView_android_layout_height,0.0f);
    int backgroud = ta.getColor(R.styleable.MyTextView_android_background, Color.BLACK);
    ta.recycle();  //注意回收
}
```

