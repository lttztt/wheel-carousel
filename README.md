## 造一个轮播的轮子

文章分三个步骤。 第一步， 实现基本功能； 第二步，考虑代码的封装性和复用性； 第三步，考虑到代码的拓展性。

### 实现基本功能

完整js代码是对应 `main.js`

```html
<div class="carousel">
  <div class="panels">
    <a href="#"><img src="img/1.jpg"></a>
    <a href="#"><img src="img/2.jpg"></a>
    <a href="#"><img src="img/3.jpg"></a>
    <a href="#"><img src="img/4.jpg"></a>
  </div>
  <div class="action">
    <span class="pre">上一个</span>
    <span class="next">下一个</span>
    <div class="dots">
      <span class="active"></span>
      <span></span>
      <span></span>
      <span></span>
    </div>
  </div>
</div>
```
```js
//让document.querySelector 用起来更方便      
const $ = s => document.querySelector(s) 
const $$ = s => document.querySelectorAll(s)

const dotCt = $('.carousel .dots')
const preBtn = $('.carousel .pre')
const nextBtn = $('.carousel .next')

//把类数组对象转换为数组，便于之后使用数组方法
//这里对应的是包含图片面板的数组
const panels = Array.from($$('.carousel .panels > a')) 
//这里对应的是包含小圆点的数组
const dots = Array.from($$('.carousel .dots span'))

//要展示第几页，就先把所有页的z-index设置为0，再把要展示的页面z-index设置为10
const showPage = pageIndex => {
  panels.forEach(panel => panel.style.zIndex = 0)
  panels[pageIndex].style.zIndex = 10
}

const setDots = pageIndex => {
  dots.forEach(dot => dot.classList.remove('active'))
  dots[pageIndex].classList.add('active')   
}

//根据第几个小点上有active的类来判断在第几页
const getIndex = () => dots.indexOf($('.carousel .dots .active'))
const getPreIndex = () => (getIndex() - 1 + dots.length) % dots.length
const getNextIndex = () => (getIndex() + 1) % dots.length

dotCt.onclick = e => {
  if(e.target.tagName !== 'SPAN') return

  let index = dots.indexOf(e.target)
  setDots(index)
  showPage(index)
}

preBtn.onclick = e => {
  let index = getPreIndex()
  setDots(index)
  showPage(index)
}

nextBtn.onclick = e => {
  let index = getNextIndex()
  setDots(index)
  showPage(index)
}

```

上面的代码使用了原生ES6语法，核心代码逻辑是：当用户点击小圆点，得到小圆点的位置（index），设置小圆点集合的样式，切换到对应的页面。页面（.panels的子元素）使用绝对定位相互重叠到一起，我们通过修改z-index把需要展示的页面放到最上层。

### 复用性和封装性

以上代码可以实现轮播基本功能，但做为意大利面条式的代码，并未做封装，无法给他人使用。另外也无法满足页面上有多个轮播的需求。


完整js代码是对应 `main3.js`

```js
class Carousel {
  constructor(root){
    this.root = root
    this.panels = Array.from(this.root.querySelectorAll('.panels a'))
    this.dots = Array.from(this.root.querySelectorAll('.dots span'))
    this.dotLength = this.dots.length
    this.dotCt = this.root.querySelector('.dots')
    this.prev = this.root.querySelector('.action .prev')
    this.next = this.root.querySelector('.action .next')
    this.bindEvent();
  }

  get index(){
    return this.dots.indexOf(this.root.querySelector('.dots span.active'));
  }
  get nextIndex(){
    return (this.index + 1) % this.dotLength
  }
  get prevIndex(){
    return (this.index - 1 + this.dotLength) % this.dotLength
  }
  bindEvent(){
  
    this.dotCt.onclick = function(e){
      if(e.target.tagName !== 'SPAN') return;
      let index = this.dots.indexOf(e.target)
      this.setDots(index)
      this.setPanels(index)
    }.bind(this)

    this.prev.onclick = e =>{
      this.setPanels(this.prevIndex)
      this.setDots(this.prevIndex)
    }
    this.next.onclick = e =>{
      this.setPanels(this.nextIndex)
      this.setDots(this.nextIndex)
    }
  }

  setDots(index){
    this.dots.forEach(el=>{
      el.classList.remove('active')
    })
    this.dots[index].classList.add('active')
  }
  setPanels(index){
    this.panels.forEach(el=>{
      el.style.zIndex = 1;
    })
    this.panels[index].style.zIndex = 10;
  }
}

var c = new Carousel(document.querySelector('.carousel'))
```

代码里用了getter，便于或者index的值。这里需要注意的是，每次调用setDot后 this.index、this.prevIndex、this.nextIndex均会自动发生变化，调用setPanels的时候需要留意。

现在轮播可以复用了，但仍有缺憾，轮播的效果太单调。假设轮播想使用fade或者slide效果，我们可以在showPage方法内修改代码。但存在的问题是效果和轮播组件做了强绑定，假设我需要另外一个效果的轮播就得新建一个组件。比如，有这样一个需求，用户可以再切页时可以随时更改效果，用上面的代码就很难办到。

能不能实现组件和效果的解绑呢？当然可以。

### 代码拓展性

设计模式中的桥接模式可以实现上述的分离。直接给代码完整效果在 `main4-animation2.js`

```js
class Carousel {
  constructor(root, animation) {
    this.animation = animation || ((to, from, onFinish) => onFinish())
    this.root = root
    ...
  }

  ...

  //showPage传递2个参数,toIndex 表示要切换到的页面(终点页面)序号,fromIndex 表示从哪个页面(起始页面)切换过来
  showPage(toIndex, fromIndex) {
    //animation函数传递3个参数，分别为终点页面dom元素，起始页面dom元素，动画执行完毕后的回调
    this.animation(this.panels[toIndex], this.panels[fromIndex], () => {
      //这里是动画执行完成后的回调          
    })
  }
}

const Animation = {
  fade(during) {
    return function(to, from, onFinish) {
      //to表示终点页面dom元素，from表示起始页面dom元素
      //对这两个元素进行适当的处理即可实现平滑过渡效果
      ...
    }
  },

  zoom(scale) {
    return function(to, from, onFinish) { /*todo...*/}
  }

}

new Carousel(document.querySelector('.carousel'), Animation.fade(300))
```

上述代码中，我们把动画类型作为参数传递给Carousel，在执行setPage的时候调用动画。 而动画函数本身做的事情比较简单：处理两个绝对定位并且相互重叠的DOM元素，以特定效果让一个元素消失另外一个元素出现。

### 动画的实现

动画可以用JS来实现（requestAnimationFrame来实现动画`main4-aimation.js`），也可以用CSS3来实现。相比JS实现动画，用CSS3性能更好并且代码更简单。

```js
const Animation = (function(){
  const css = (node, styles) => Object.entries(styles)
  .forEach(([key, value]) => node.style[key] = value)
  const reset = node => node.style = ''

  return {
    fade(during = 400) {
      return function(to, from, onFinish) {
        css(from, {
          opacity: 1,
          transition: `all ${during/1000}s`,
          zIndex: 10
        })
        css(to, {
          opacity: 0,
          transition: `all ${during/1000}s`,
          zIndex: 9
        })

        setTimeout(() => {
          css(from, {
            opacity: 0,
          })
          css(to, {
            opacity: 1,
          })              
        }, 100)

        setTimeout(() => {
          reset(from)
          reset(to)
          onFinish && onFinish()
        }, during)

      }
    },

    zoom(scale = 5, during = 600) {
      return function(to, from, onFinish) {
        css(from, {
          opacity: 1,
          transform: `scale(1)`,
          transition: `all ${during/1000}s`,
          zIndex: 10
        })
        css(to, {
          zIndex: 9
        })

        setTimeout(() => {
          css(from, {
            opacity: 0,
            transform: `scale(${scale})`
          })             
        }, 100)

        setTimeout(() => {
          reset(from)
          reset(to)
          onFinish && onFinish()
        }, during)

      }
    }
  }
})()
```

### 总结

[效果预览](http://js.jirengu.com/cajet)