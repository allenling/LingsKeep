Polymer 2.0的一些问题
========================


app-route
----------


https://polymer.slack.com/archives/C03PF4L4L/p1497384792282133

https://polymer.slack.com/archives/C03PF4L4L/p1497385393480519

https://polymer.slack.com/archives/C03PF4L4L/p1497542122128701

https://polymer.slack.com/archives/C03PF4L4L/p1497553631212096


使用polymer-starter-kit 2.0的时候，在url上加query string之后，query string会消失，我去slack问了下为什么

slack中polymer的开发者(目测)回答说是app-route/app-location的bug，还不确定，然后解决方式是在app的construct方法中加入


.. code-block::

    Polymer.RenderStatus.beforeNextRender(this, (qs) => this.$.location.__query = qs, [location.search.slice(1)]);

上面这段代码

Polymer.RenderStatus.beforeNextRender的传参是
(context, function, args)

也就是是在页面渲染之前的调用我们指定的回调函数，调用一次就将函数加入到回调的queue中，然后从queue中pop出来

但是这样url其实变了两次，一次从/myview?pk=1变为/mywei，这里吃掉了query string，这里走正常的routeData.page的change事件，也就是会走到psk中的_pageChanged

然后第二次又变为/myview?pk=1，不知道谁把query string又上到url上了，然后由于page没有变化，所以不会走正常的routeData.page的change事件，也就是不会走到psk中的_pageChanged

所以my-app中如果按照psk中的例子的话，在_pageChanged中我们需要拿到queryParams的话，就不能简单的将上面的代码块加入到construct函数中，我们可以加入一个回调，回调函数u我们定义的_pageChange，这样

我们只会在最后query string补全之后进行page的匹配，注意的是要把_pageChange从paged的observer中去掉，不然第一次url变化会调用_pageChanged，然后qs的回调有走一次_pageChanged。


.. code-block::

    constructor() {
      super();

      this.rootPattern = (new URL(this.rootPath)).pathname;
      Polymer.RenderStatus.beforeNextRender(
        this, (qs) => this.$.location.__query = qs, [location.search.slice(1)]
      );
      Polymer.RenderStatus.beforeNextRender(
        this, function () {
          this._pageChanged(this.page);
        }, []
      );
    }



