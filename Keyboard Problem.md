## Fragment
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {  
    super.onViewCreated(view, savedInstanceState)  
  
    ViewCompat.setOnApplyWindowInsetsListener(scrollView) { v, insets ->  
        val imeInsets = insets.getInsets(WindowInsetsCompat.Type.ime())  
        // apply bottom padding equal to IME height (preserve original paddings if needed)  
        v.setPadding(v.paddingLeft, v.paddingTop, v.paddingRight, imeInsets.bottom)  
        // return insets so other listeners can use them  
        insets  
    }  
  
    ViewCompat.requestApplyInsets(scrollView)  
    val extraGapPx = TypedValue.applyDimension(  
        TypedValue.COMPLEX_UNIT_DIP, 8f, resources.displayMetrics  
    ).toInt()  
    fun scrollCaretIntoView() {  
        val layout = noteText.layout ?: return  
        val sel = noteText.selectionStart.coerceAtLeast(0)  
        val line = layout.getLineForOffset(sel)  
        val lineTop = layout.getLineTop(line)  
        val lineBottom = layout.getLineBottom(line)  
        val primaryHorizontal = layout.getPrimaryHorizontal(sel).toInt()  
  
        // Make a small rectangle around the caret / current line in editText coordinates  
        val left = (primaryHorizontal - 10).coerceAtLeast(0)  
        val top = (lineTop - extraGapPx).coerceAtLeast(0)  
        val right = left + 20  
        val bottom = lineBottom + extraGapPx  
  
        val rect = Rect(left, top, right, bottom)  
  
        // requestRectangleOnScreen expects coordinates relative to the EditText itself.  
        // Post to ensure layout has settled (important when called from text watcher)        noteText.post {  
            // This asks the nearest scrolling parent (NestedScrollView) to make rect visible.  
            noteText.requestRectangleOnScreen(rect, /* immediate = */ true)  
        }  
    }  
    noteText.addTextChangedListener(object : TextWatcher {  
        override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}  
        override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {}  
        override fun afterTextChanged(s: Editable?) {  
            scrollCaretIntoView()  
        }  
    })  
    noteText.onSelectionChangedListener = { _, _ ->  
        scrollCaretIntoView()  
    }  
    noteText.setOnTouchListener { v, event ->  
        if (event.action == MotionEvent.ACTION_UP) {  
            v.performClick() // accessibility  
            noteText.post { scrollCaretIntoView() }  
        }  
        // Let the EditText handle focus/selection/etc.  
        false  
    }  
}
```

## XML File
```xml
<androidx.core.widget.NestedScrollView  
    android:id="@+id/scrollView"  
    android:layout_width="match_parent"  
    android:layout_height="0dp"  
    android:fillViewport="true"  
    app:layout_constraintBottom_toBottomOf="parent"  
    app:layout_constraintEnd_toEndOf="parent"  
    app:layout_constraintStart_toStartOf="parent"  
    app:layout_constraintTop_toBottomOf="@id/edit_mode_appbar">  
  
    <!-- If you want the EditText to grow, use wrap_content -->  
    <LinearLayout  
        android:layout_marginTop="20dp"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:background="@null"  
        android:orientation="vertical"  
        android:paddingStart="20dp"  
        android:paddingEnd="20dp">  
  
        <EditText  
            android:id="@+id/note_title"  
            android:layout_width="match_parent"  
            android:layout_height="wrap_content"  
  
            android:background="@null"  
            android:hint="@string/title"  
            android:textColor="@color/white"  
            android:textColorHint="@color/hint"  
            android:textSize="@dimen/title_text_size">  
  
        </EditText>  
        <com.example.noteapp.CustomEditText            android:id="@+id/note_text"  
            android:layout_width="match_parent"  
            android:layout_height="wrap_content"  
            android:background="@null"  
            android:gravity="top"  
            android:hint="@string/note_hint"  
            android:inputType="textMultiLine"  
            android:minLines="5"  
            android:scrollbars="vertical"  
            android:textColor="@color/white"  
            android:textColorHint="@color/hint"  
            android:textSize="@dimen/note_text_size" />  
    </LinearLayout>  
</androidx.core.widget.NestedScrollView>
```

## Custom Edit Text
```kotlin

import android.content.Context  
import android.util.AttributeSet  
import androidx.appcompat.widget.AppCompatEditText  
  
class CustomEditText @JvmOverloads constructor(  
    context: Context,  
    attrs: AttributeSet? = null,  
    defStyleAttr: Int = android.R.attr.editTextStyle  
) : AppCompatEditText(context, attrs, defStyleAttr) {  
  
    /**  
     * Called when selection/cursor changes.     * start = selectionStart, end = selectionEnd     */    var onSelectionChangedListener: ((start: Int, end: Int) -> Unit)? = null  
  
    override fun onSelectionChanged(selStart: Int, selEnd: Int) {  
        super.onSelectionChanged(selStart, selEnd)  
        onSelectionChangedListener?.invoke(selStart, selEnd)  
    }  
  
    // IMPORTANT for accessibility/lint: implement performClick()  
    override fun performClick(): Boolean {  
        // let the system handle click-related accessibility events  
        return super.performClick()  
    }  
}
```

