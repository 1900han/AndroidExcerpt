为了以后能少Google自定义Dialog这些基本的套路,现在把相关的步骤记录下来
```java
public class CustomertDialog extends Dialog {
    private final Context context;
    private String title;
    private List<String> itemList;
    private TextView titleView;
    private LinearLayout llayoutCustomerContainer;
    private ICustomDialogEventListener onCustomDialogEventListener;

    public CustomerInfoSelectDialog(Context context, int themeResId, String title, List<String> itemList,
                                    ICustomDialogEventListener onCustomDialogEventListener) {
        super(context, themeResId);
        this.context = context;
        this.itemList = itemList;
        this.title = title;
        this.onCustomDialogEventListener = onCustomDialogEventListener;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        LayoutInflater inflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        View layout = inflater.inflate(R.layout.dialog_customer_info, null);
        setContentView(layout);
        this.setCancelable(false);
        llayoutCustomerContainer = (LinearLayout) layout.findViewById(R.id.llayout_customer_container);
        titleView = (TextView) layout.findViewById(R.id.txt_dialog_customer_title);
        setData();
    }

    private void setData() {
        titleView.setText(title);

        final String[] arr = itemList.toArray(new String[0]);
        if (arr.length > 0) {
            for (int i = 0; i < arr.length; i++) {
                ViewGroup.LayoutParams tlp = new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, DisplayUtil.dip2px(context, 50));
                final TextView textView = new TextView(context);
                textView.setLayoutParams(tlp);
                textView.setText(arr[i]);
                textView.setTextColor(context.getResources().getColor(R.color.text_black_2F2F2F));
                textView.setTextSize(14);
                textView.setGravity(Gravity.CENTER_VERTICAL);
                textView.setPadding(DisplayUtil.dip2px(context, 20), 0, 0, 0);
                textView.setOnClickListener(new View.OnClickListener() {
                    @Override
                    public void onClick(View v) {
                        onCustomDialogEventListener.customDialogEvent(textView.getText().toString());
                        dismiss();
                    }
                });
                llayoutCustomerContainer.addView(textView);
                if (i != arr.length - 1) {
                    ViewGroup.LayoutParams vlp = new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, DisplayUtil.dip2px(context, 1));
                    View view = new View(context);
                    view.setLayoutParams(vlp);
                    view.setBackgroundColor(context.getResources().getColor(R.color.line_e5e5e5));
                    llayoutCustomerContainer.addView(view);
                }
            }
        }
    }

    // 利用interface来构造一个回调函数
    public interface ICustomDialogEventListener {
        public void customDialogEvent(String valueSendBackToTheActivity);
    }
}
```
对应的dialog布局
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/llayout_customer_container"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@drawable/bg_customer_dialog"
        >
    <TextView
            android:id="@+id/txt_dialog_customer_title"
            android:layout_width="match_parent"
            android:layout_height="@dimen/y60"
            android:textSize="@dimen/x14"
            android:gravity="center"
            android:textColor="@color/text_gray_999999"
            />
    <View
            android:layout_width="match_parent"
            android:layout_height="@dimen/x1"
            android:background="@color/line_e5e5e5"
            />
</LinearLayout>
```
如果dialog需要圆角
```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <corners android:radius="4dp"/>
    <stroke android:color="@color/bg_gray_d4d4d4"/>
    <solid android:color="@color/bg_white"/>
</shape>
```
Dialog的Theme
```xml
<style name="AlertDialogCustom" parent="@android:style/Theme.Dialog">
        <item name="android:windowFrame">@null</item>
        <item name="android:windowIsFloating">true</item>
        <item name="android:windowContentOverlay">@null</item>
        <item name="android:windowNoTitle">true</item>
        <item name="android:background">@color/android:transparent</item>
        <!--去除对话框四周阴影-->
        <item name="android:windowBackground">@null</item>
    </style>
```

