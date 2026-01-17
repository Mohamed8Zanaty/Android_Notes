## Pick Image Launcher
```kotlin
 var imagePath : String? = null  
 val pickImageLauncher =  
    registerForActivityResult(ActivityResultContracts.GetContent()) { uri: Uri? ->  
        uri?.let {  
  
            imagePath = getPathFromUri(it)  
        }  
    }
```

## Convert From URI to Path String
```kotlin
private fun getPathFromUri(uri: Uri): String? {  
   var filePath: String? = null  
   val cursor = contentResolver.query(uri, null, null, null, null)  
   cursor?.use {  
       if (it.moveToFirst()) {  
           val columnIndex = it.getColumnIndexOrThrow("_data")  
           filePath = it.getString(columnIndex)  
       }  
   }  
   return filePath ?: uri.toString()  
}
```

## Select Image Button
```kotlin
binding.btnSelectImage.setOnClickListener {  
    pickImageLauncher.launch("image/*")  
}
```

## In Adapter
```kotlin
val bitmap: Bitmap? = BitmapFactory.decodeFile(item.image)  
binding.imageView.setImageBitmap(bitmap)
```
