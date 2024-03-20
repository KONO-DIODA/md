# Vue2 和 vue3 的区别 #
**区别 1：响应式原理**  
**Vue2** 的双向数据绑定使用了 ES5 中的 API：Object.defineProperty()  
    
    const a = {}
    let bValue = 1
    Object.defineProperty(a, "b", {
      enumerable: false,
      // 表示是否可以通过for..in..遍历出来，默认值为false（输出对象时也不会展示出来）
      // 一般通过new基本包装类型(Object, Array, Number)等对象创建的实例对象，无法通过for..in..遍历
      // 而通过如Array.prototype这种方式添加的属性方法可以被for..in..遍历，但无法被Object.keys()和JSON.stringfy()获取
      // 可枚举性还会影响Object.keys()和JSON.stringfy()
      // Object.keys()会返回一个由key构成的数组，JSON.stringfy()会返回对象的字符串化表示
      writable: true,
      // 表示该属性是否可被更改，默认值为false
      configurable: true,
      // 表示该属性是否可被删除，默认值为false
      value: bValue,
    })
    // 属性定义value和监听定义不能写在一起
    Object.defineProperty(a, "b", {
      get:function() {
        console.log(bValue)
        return bValue
      },
      set:function(value) {
        bValue = value
        console.log("setted")
      }
    })

考虑到对象的深层监听，需要对以上代码进行优化

    function Observer(data) {
      this.walk(data)
    }
    let p = Observer.prototype
    p.walk = function(obj) {
      let val = undefined
      for(let key in obj) {
        if(obj.hasOwnProperty(key)) {
          val = obj[key]
          if(typeof val === 'object') {
            new Observer(val)
          }
          this.convert(obj, key, val)
        }
      }
    }
	p.convert = function(obj, key, val) {
	  Object.defineProperty(obj, key, {
	    enumerable: false,
		configurable: false,
		get: function() {
		console.log(key + '被访问')
		return val //此处如果不返回值会导致调用时获取为空
		},
		set: function(newVal) {
		console.log(key + '被修改')
		if(newVal === val) return
		val = newVal
		}
	  })
	}
	let data = {
	  user: {
		name: 'bai',
		age: 18
	  },
	  city: 'nanjing'
	}
	let app = new Observer(data) // user 被访问
	console.log(typeof [] === 'object') // 数组也是对象

但是以上方法无法实现对数组操作的监听  
可以定义一个方法对象，对象中的 key 为数组的各个方法名称，每个key 的值为一个函数，函数中先进行监听操作，再调用 Array.prototype
中的方法。将数组的\_proto\_设置为该方法对象后，数组调用相应方
法时就会经由这个函数去调用原先的方法。（或者直接对原型中的对
应方法进行改写）  
如需对对象的新增属性和删除属性进行监听，需令其强制调用己方定
义的函数进行增添或删除操作  

	const a = {
	  a: 1,
	  b: 2,
	  delete: function(key) {
		console.log(key + '被删除了')
		delete a[key]
	  }
	}
	a.delete("a")
	console.log(a)

**Vue3** 中使用 Proxy-Reflect 实现响应式  

	let person = {
	  name: '张三',
	  age: 18
	}
	//模拟 vue3 中实现响应式
	const p = new Proxy( person, {
	  get(target, proName) {
		console.log(`有人读取了 p 身上的${proName}属性`)
		return Reflect.get(target, proName)
	  },
	  // 在新增属性时，也会调用 set
	  set( target, proName, value ) {
		console.log(`有人修改了 p 身上的${proName}属性,更新页面！`)
		return Reflect.set( target, proName, value )
	  },
	  deleteProperty( target, proName ) {
		console.log(`有人删除了 p 身上的${proName}属性,更新页面！`)
		return Reflect.deleteProperty(target, proName) //返回一个标识删除是否成功的布尔值
	  }
	})

ref、reactive 的原理，在创建响应式数据的时候返回一个 proxy 对象，在对该对象进行 get 操作时，会进行依赖收集，然后在对该对象的属性进行变更时会将之前收集到的有关该属性的依赖全部触发一次  
