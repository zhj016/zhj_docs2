#Android属性之build.prop生成过程分析

build.prop的生成是由make系统解析build/core/Makefile完成。

1)      Makefile中首先定义各种变量，这在下一步执行时会用到。比如：

    ...  
    PRODUCT_DEFAULT_LANGUAGE="$(calldefault-locale-language,$(PRODUCT_LOCALES))" \  
    PRODUCT_DEFAULT_REGION="$(calldefault-locale-region,$(PRODUCT_LOCALES))" \  
    ...

2)      Makefile中调用build/tools/buildinfo.sh执行脚本，并输出到build.prop

Buildinfo.sh很简单，只是echo一些属性，比如：

  echo"ro.product.locale.language=$PRODUCT_DEFAULT_LANGUAGE"  
  echo"ro.product.locale.region=$PRODUCT_DEFAULT_REGION"  

3)      Makefile中直接把$(TARGET_DEVICE_DIR)/system.prop的内容追加到build.prop中。

4)      收集ADDITIONAL_BUILD_PROPERTIES中的属性，追加到build.prop中。

ADDITIONAL_BUILD_PROPERTIES又会收集PRODUCT_PROPERTY_OVERRIDES中定义的属性

      ADDITIONAL_BUILD_PROPERTIES:= \  
              $(ADDITIONAL_BUILD_PROPERTIES)\  
              $(PRODUCT_PROPERTY_OVERRIDES)  


通过build.prop生成过程的分析，可知哪里可以修改原有的属性或加入自己定义属性，那就是2) buildinfo.sh; 3) system.prop; 4) ADDITIONAL_BUILD_PROPERTIES或PRODUCT_PROPERTY_OVERRIDES。不过个人建议改在system.prop或PRODUCT_PROPERTY_OVERRIDES，这对应于具体特定平台或产品的修改。
