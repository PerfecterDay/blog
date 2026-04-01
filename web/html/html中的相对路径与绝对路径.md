
# html中的相对路径与绝对路径
{docsify-updated}



# html中的相对路径与绝对路径
------------------------------
![相对路径与绝对路径](../pics/abslout-relative.gif)

|引用者|      被引用者|     相对路径|                        绝对路径|
|------|      ------|      --------|                        -------|
|Ref1.htm|    BeRef1.gif|  ../SubDir2/BeRef1.gif|        /Dir1/SubDir2/BeRef1.gif|
|Ref2.htm|    BeRef1.gif| ../../Dir1/SubDir2/<br/>BeRef1.gif| /Dir1/SubDir2/ BeRef1.gif|
|Ref1.htm|    BeRef2.htm|   ../../Dir2/BeRef2.htm|           /Dir2/BeRef2.htm|
|Ref2.htm|    BeRef2.htm|   ../BeRef2.htm|                   /Dir2/BeRef2.htm|

