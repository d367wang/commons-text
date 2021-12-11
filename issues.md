#### src/main/java/org/apache/commons/text/translate/UnicodeUnescaper.java

```
@Override
public int translate(final CharSequence input, final int index, final Writer out) throws IOException {
   if (input.charAt(index) == '\\' && index + 1 < input.length() && input.charAt(index + 1) == 'u') {
       // consume optional additional 'u' chars
       int i = 2;
       while (index + i < input.length() && input.charAt(index + i) == 'u') {
           i++;
       }

       if (index + i < input.length() && input.charAt(index + i) == '+') {
           i++;
       }

       if (index + i + 4 <= input.length()) {
           // Get 4 hex digits
           final CharSequence unicode = input.subSequence(index + i, index + i + 4);

           try {
               final int value = Integer.parseInt(unicode.toString(), 16);
               out.write((char) value);   
           } catch (final NumberFormatException nfe) {
               throw new IllegalArgumentException("Unable to parse unicode value: " + unicode, nfe);
           }
           return i + 4;
       }
       throw new IllegalArgumentException("Less than 4 hex digits in unicode value: '"
               + input.subSequence(index, input.length())
               + "' due to end of CharSequence");
   }
   return 0;
}
```

In the try block, a string of 4 characters are parsed to an integer `value`, and converted to `char`. 
When the input is parsed to a negative integer (e.g. `input` is `\\u-FFF`), this is unsafe.


&nbsp;

&nbsp;

#### src/main/java/org/apache/commons/text/AlphabetConverter.java

```
private static String codePointToString(final int i) {
   if (Character.charCount(i) == 1) { 
       return String.valueOf((char) i);         
   }
   return new String(Character.toChars(i));
}
```

The JDK method `Character.charCount` doesn't validate the specified character to be a valid Unicode code point, and it returns 1 when i < 0. So when the method `codePointToString` is passed an negative integer, the if-condition is true. Then `i` is converted to `char` and passed to `String.valueOf`.
This method is called in the following method.
```
public static AlphabetConverter createConverterFromMap(
       final Map<Integer, String> originalToEncoded) {
   final Map<Integer, String> unmodifiableOriginalToEncoded =
           Collections.unmodifiableMap(originalToEncoded);
   final Map<String, String> encodedToOriginal = new LinkedHashMap<>();

   int encodedLetterLength = 1;

   for (final Entry<Integer, String> e
           : unmodifiableOriginalToEncoded.entrySet()) {
       final String originalAsString = codePointToString(e.getKey());    
       encodedToOriginal.put(e.getValue(), originalAsString);

       if (e.getValue().length() > encodedLetterLength) {
           encodedLetterLength = e.getValue().length();
       }
   }

   return new AlphabetConverter(unmodifiableOriginalToEncoded,
           encodedToOriginal,
           encodedLetterLength);
}

```
An input that may have unexpected behavior is a HashMap that contains K-V pair (-1, 'x')



&nbsp;

&nbsp;


#### src/main/java/org/apache/commons/text/StrBuilder.java
```
       @Override
       public int read(final char[] b, final int off, int len) {
           if (off < 0 || len < 0 || off > b.length
                   || (off + len) > b.length || (off + len) < 0) {
               throw new IndexOutOfBoundsException();
           }
           if (len == 0) {
               return 0;
           }
           if (pos >= StrBuilder.this.size()) {
               return -1;
           }
           if (pos + len > size()) {
                len = StrBuilder.this.size() - pos;   // false positive
           }
           StrBuilder.this.getChars(pos, pos + len, b, off);
           pos += len;
           return len;
       }
```
In the if-condition, `(off + len) < 0` is redundant. 