### ViewPager中常用两种adapter区别

* **FragmentPagerAdapter**

  每页都是一个Fragment，并且所有的Fragment实例一直保存在Fragment manager中。所以它适用于**少量**固定的fragment，比如一组用于分页显示的标签。除了当Fragment不可见时，它的**视图层**(view hierarchy)有可能被**销毁**外，**每页**的**Fragment**都会被**保存在内存**中。(翻译自代码文件的注释部分)

* **FragmentStatePagerAdapter**

  每页都是一个Fragment，当Fragment不被需要时（比如**不可见**），**整个Fragment都会被销毁**，除了saved state被保存外（保存下来的bundle用于恢复Fragment实例）。所以它适用于**很多页**的情况。（翻译自代码文件的注释部分）

