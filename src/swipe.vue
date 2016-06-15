<template>
  <div class="wrap">
    <div class="container" @touchstart="touchStart" @touchmove="touchMove" @touchend="touchEnd" v-el:container>
      <slot></slot>
    </div>
    <div class="indicatorContainer" v-show="showIndicators">
      <div class="indicator" v-for="page in pages" :class="{ active: $index === index }"></div>
    </div>
  </div>
</template>
<style scoped>
  .wrap {
    width: 100%;
    position: relative;
    overflow: hidden;
  }

  .container {
    height: 100%;
    width: 300%;
    position: relative;
    left: -100%;
    background: red;
  }

  .indicatorContainer {
    position: absolute;
    opacity: .3;
    bottom: 10px;
    left: 50%;
    transform: translateX(-50%);
  }

  .indicator {
    border-radius: 50%;
    width: 8px;
    height: 8px;
    float: left;
    margin: 0 3px;
    background: #000000;
  }

  .indicator.active {
    background: #ffffff;
  }
</style>
<script type="text/ecmascript-6">
  import { once, addClass, removeClass } from 'wind-dom';

  export default{
    props: {
      speed: {
        type: Number,
        default: 300
      },
      auto: {
        type: Number,
        default: 3000
      },
      showIndicators: {
        type: Boolean,
        default: true
      },
      prevent: {
        type: Boolean,
        default: false
      }
    },
    events: {
      // 当子组件个数改变时重新init
      swipeItemCreated(){
        if( this.childrenCount !== this.$children.length ){
          this.init()
        }
      },
      swipeItemDestroyed(){
        if( this.childrenCount !== this.$children.length ){
          this.init()
        }
      }
    },
    data(){
      return {
        index: 0,
        pages: [],
        leftIndex: null,
        current: null,
        rightIndex: null,
        clientWidth: 0,
        container: null,
        timer: null,
        prefix: 0,
        childrenCount:0,
        dragState: {
          startClientX: 0,
          startClietnY: 0,
          endOffsetX: 0,
          endOffsetY: 0,
          startTime: 0,
          onDrag: false,
          onAnimate: false,
          verticalScrolling: false
        },
        // 当swipeItem只有两个时，需要用到这两个临时元素，使用两个字段保存是为了防止过多的使用cloneNode方法，看setPagePostion方法
        tmpEl1:null,
        tmpEl2:null
      }
    },
    methods: {
      init(){
        this.childrenCount = this.$children.length
        this.container = this.$els.container
        // 在外部，当容器的子组件动态增加删除时，vue为了复用并不会真的把dom删除而是隐藏它
        // 这时候我们必须手动把该容器下的所有div元素的display全部初始化为‘none’
        Array.prototype.slice.call(this.container.children,0).map( item => {
          item.style.display = 'none'
        })
        //不能使用 this.pages = this.container.children，因为之后我们会对container做insert操作，会导致this.pages变化，
        //而这里我们并不希望this.pages再出现动态的增加或者减少
        this.pages = this.$children.map( item => {
          return item.$el
        })
        if ( this.pages.length === 0 )return
        else if ( this.pages.length === 1 ) {
          this.current = this.pages[0]
          this.current.style.display = 'block'
          this.current.style.transform = this.current.style.webkitTransform = ''
          this.container.style.left = '0'
          this.showIndicators = false
          clearInterval( this.timer )
        } else {
          this.container.style.left = '-100%'
          this.showIndicators = true
          if( this.pages.length === 2 ) {
            // 当只有两个子组件时，我们需要额外补充两个克隆组件才能填充长度为300%的容器，在setPagePosition会做处理
            this.tmpEl1 = this.pages[0].cloneNode( true )
            this.tmpEl2 = this.pages[1].cloneNode( true )
            this.container.insertBefore( this.tmpEl1 , this.container.firstChild )
            this.container.insertBefore( this.tmpEl2 , this.container.firstChild )
          }
          this.clientWidth = this.$el.clientWidth
          this.setPagePosition( 0 )
          clearInterval( this.timer )
          this.autoScroll()
        }
      },
      setPagePosition(index, direction){
        if ( direction === 'showPrev' ) {
          this.right.style.display = 'none'
        } else if ( direction === 'showNext' ) {
          this.left.style.display = 'none'
        }
        this.index = index
        if ( index === 0 ) {
          this.leftIndex = this.pages.length - 1
        } else {
          this.leftIndex = index - 1
        }
        if ( index === this.pages.length - 1 ) {
          this.rightIndex = 0
        } else {
          this.rightIndex = index + 1
        }
        this.left = this.pages[ this.leftIndex ]
        this.current = this.pages[ this.index ]
        this.right = this.pages[ this.rightIndex ]
        // 处理当只有2个切换项时的问题
        if( this.pages.length === 2 ) {
          this.tmpEl2.style.display = 'none'
          this.tmpEl1.style.display = 'none'
          if( index === 0 ){
            this.left = this.tmpEl2
          }else if( index === 1 ){
            this.left = this.tmpEl1
          }
        }
        this.left.style.transform = 'translateX(0)'
        this.left.style.display = 'block'
        this.current.style.transform = 'translateX(100%)'
        this.current.style.display = 'block'
        this.right.style.transform = 'translateX(200%)'
        this.right.style.display = 'block'
      },
      touchStart(e){
        if( this.pages.length < 2 ) return

        if ( this.prevent ) {
          e.preventDefault();
        }
        clearInterval(this.timer)
        /*
         * 时间放在这里是有讲究的,如果放在return语句的下面,就会遇到,touchstart时this.dragState.onAnimate为true,没有重新获得新时间
         * 而endtouch时this.dragState.onAnimate刚好为false,这时候去比对时间差,这样会导致这时间差变大
         * */
        this.dragState.startTime = new Date()
        if ( this.dragState.onAnimate )return
        var touches = e.touches[ 0 ]
        this.dragState.startClientX = touches.clientX
        this.dragState.startClientY = touches.clientY
      },
      touchMove(e){
        if( this.pages.length < 2 ) return
        /*
         * 0.当上下移动距离小于8左右移动距离小于10时,我们不为所动
         * 1.当手指上下移动距离超过10px,则此次的所有操作都不会触发左右滚动 (需要额外全局参数,this.verticalScrolling)
         * 2.当收拾左右移动距离超过10px,则此次操作只会导致左右滚动而不会出现上下滚动,由于左右移动是超过10px时才会触发,所以真实
         * 移动距离得处理这10px的误差,当左移动时,减去10px,当右移动时加上10px
         * */
        if ( this.dragState.onAnimate ) {
          e.preventDefault()
          return
        }
        var touches = e.touches[ 0 ]
        this.dragState.endOffsetX = touches.clientX - this.dragState.startClientX
        var threshold = 5

        if ( this.dragState.onDrag ) {
//          防止默认行为,阻止它操作上下移动
          e.preventDefault()
//          阻止冒泡,比如点击,拖动等事件不会传递到它上层元素
          e.cancelBubble = true
          this.translate(this.dragState.endOffsetX - this.prefix)
        } else if ( Math.abs(this.dragState.endOffsetX) > threshold && !this.dragState.verticalScrolling ) {
          e.preventDefault()
          this.dragState.onDrag = true
          this.dragState.verticalScrolling = false
          this.prefix = this.dragState.endOffsetX > 0 ? threshold : -threshold
        }
//        这里this.dragState.verticalScrolling 存在的意义在于,当此次滚动中出现上下滚动,那直到手指离开前,都不会触发左右滚动
        else if ( Math.abs(touches.clientY - this.dragState.startClientY) > threshold && !this.dragState.onDrag || this.dragState.verticalScrolling ) {
          this.dragState.onDrag = false
          this.dragState.verticalScrolling = true
        }
      },
      touchEnd(){
        if ( this.dragState.onAnimate || this.pages.length < 2 ) return
        this.dragState.verticalScrolling = false
        /*
         * 这么做是为了当快速切换的时候,this.dragState.startTime 还没来记得触发touchstart事件初始化就开始触发touchend,导致
         * 时间参数还是上一次的参数,此时再去相减的话就会造成时间差变长
         * */
        var interval = new Date() - this.dragState.startTime
        if ( this.dragState.endOffsetX !== 0 && this.dragState.onDrag ) {
          this.dragState.onAnimate = true
          this.dragState.onDrag = false
          if ( Math.abs(this.dragState.endOffsetX ) > this.clientWidth / 2 || (Math.abs( this.dragState.endOffsetX ) > 20 && interval < 500 ) ) {
            if ( this.dragState.endOffsetX > 0 ) {
              this.translate(this.clientWidth, true, ()=> {
                this.setPagePosition(this.leftIndex, 'showPrev')
              })
            } else {
              this.translate(-this.clientWidth, true, ()=> {
                this.setPagePosition(this.rightIndex, 'showNext')
              })
            }
          } else {
            this.translate(0, true)
          }
//        等待理论上的动画时间结束后才开始
          setTimeout(()=> {
            this.autoScroll()
          }, this.speed)
        } else {
          this.autoScroll()
        }
        this.dragState.endOffsetX = 0
      },
      translate(offset, auto, callback){
        var element = this.container
        if ( auto ) {
          element.style.webkitTransition = 'transform ' + this.speed + 'ms ease'
          setTimeout(() => element.style.webkitTransform = `translate3d(${offset}px, 0, 0)`, 50)
          var called = false
          var transitionEndCallback = () => {
            if ( called ) return
            called = true
            element.style.webkitTransition = ''
            element.style.webkitTransform = ''
            if ( callback ) {
              callback.apply(this, arguments)
            }
            this.dragState.onAnimate = false
          }
          once(element, 'webkitTransitionEnd', transitionEndCallback)
          setTimeout(transitionEndCallback, this.speed + 100) // webkitTransitionEnd maybe not fire on lower version android.
        } else {
          element.style.webkitTransition = ''
          element.style.transform = `translate3d(${offset}px,0,0)`
        }
      },
      autoScroll(){
        clearInterval(this.timer)
        if ( this.auto > 0 && this.pages.length > 1 ) {
          this.timer = setInterval(()=> {
            this.translate(-this.clientWidth, true, ()=> {
              this.setPagePosition(this.rightIndex, 'showNext')
            })
          }, this.auto)
        }
      }
    }
  }
</script>
