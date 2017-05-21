// 同 jquery 一样，underscore 的整体也是包裹在一个闭包当作，这样可以避免污染全局变量
// 最后通过传入this（window）来改变函数的作用域
(function () {


    // 建立一个根元素
    var root = this;


    // 然后保存 _ 这个变量（防止冲突）
    // 和下面的 _.noConflict 方法配合使用
    var previousUnderscore = root._;



    // 之所有保存原型对象，是因为在压缩的时候
    // 类似像 prototype 这样的名称修改后浏览器就不认了（比如会被压缩成变量 a）
    var ArrayProto = Array.prototype,
        ObjProto = Object.prototype,
        FuncProto = Function.prototype;



    // 缓存变量
    var
        push = ArrayProto.push,
        slice = ArrayProto.slice,
        toString = ObjProto.toString,
        hasOwnProperty = ObjProto.hasOwnProperty;



    // 保存原生方法
    var
        nativeIsArray = Array.isArray,
        nativeKeys = Object.keys,
        nativeBind = FuncProto.bind,
        nativeCreate = Object.create;



    // 用于代理原型转换的空函数
    // 主要是处理 Object.create()
    // 因为原型无法直接实例化
    // 这个是和下面的 baseCreate 配合使用
    var Ctor = function () { };



    // 初始化
    // 如果参数是 underscore 的一个实例，就直接返回该参数
    // 否则的话，返回一个实例化之后的对象
    // 都没有的话就将该实例保存
    var _ = function (obj) {
        if (obj instanceof _) return obj;
        if (!(this instanceof _)) return new _(obj);
        this._wrapped = obj;
    };



    // 根据环境来将 underscore 的命名变量存放到不同的对象中
    if (typeof exports !== 'undefined') {
        // node.js 中才有 exports 和 module.exports 方法
        // 即在 node.js 环境下将 underscore 作为一个模块使用，并向后兼容旧版的模块 API
        if (typeof module !== 'undefined' && module.exports) {
            exports = module.exports = _;
        }
        exports._ = _;
    // 否则就是浏览器环境
    // 将 underscore 以 _ 暴露到全局
    } else {
        root._ = _;
    }



    // 版本信息
    _.VERSION = '1.8.2';




    // 这里主要用来执行函数并改变所执行函数的作用域
    // 对正常传入的函数进行了一层包装处理，以便重复利用，也保证了 this 的上下文
    var optimizeCb = function (func, context, argCount) {

        // void 0 返回 undefined，即没有传入上下文信息的时候直接返回相应的函数
        if (context === void 0) return func;


        // 对参数个数小于等于4的情况进行分类处理
        switch (argCount == null ? 3 : argCount) {


            // 1 个参数，一般是用在接受单值的情况，比如 times，sortedIndex 之类的函数
            // 2 个参数，网上搜索据说是给 jQuery，Zepto 事件绑定代理之类，源码中没有找到相关使用
            // 3 个参数，适用于迭代器，比如 forEach，map，pick 等
            // 4 个参数，reduce 和 reduceRight 函数
            case 1:
                return function (value) {
                    return func.call(context, value);
                };
            case 2:
                return function (value, other) {
                    return func.call(context, value, other);
                };
            case 3:
                return function (value, index, collection) {
                    return func.call(context, value, index, collection);
                };
            case 4:
                return function (accumulator, value, index, collection) {
                    return func.call(context, accumulator, value, index, collection);
                };
        }


        // 如果以上均不符合，则直接使用 apply 调用相关函数
        return function () {
            return func.apply(context, arguments);
        };
    };


    // 针对集合迭代的回调处理
    var cb = function (value, context, argCount) {


        // 如果不传 value，返回等价的自身
        // 如果传入函数，返回该函数的回调
        // 如果传入对象，寻找匹配的属性值
        // 如果都不是，返回相应的属性访问器
        if (value == null) return _.identity;
        if (_.isFunction(value)) return optimizeCb(value, context, argCount);
        if (_.isObject(value)) return _.matcher(value);
        return _.property(value);
    };

    // 默认的迭代器，返回 cb 函数
    _.iteratee = function (value, context) {
        return cb(value, context, Infinity);
    };



    // 待续

    var createAssigner = function (keysFunc, undefinedOnly) {
        return function (obj) {
            var length = arguments.length;
            if (length < 2 || obj == null) return obj;
            for (var index = 1; index < length; index++) {
                var source = arguments[index],
                    keys = keysFunc(source),
                    l = keys.length;
                for (var i = 0; i < l; i++) {
                    var key = keys[i];
                    if (!undefinedOnly || obj[key] === void 0) obj[key] = source[key];
                }
            }
            return obj;
        };
    };

    var baseCreate = function (prototype) {

        // 如果参数不是对象，直接返回空对象
        // 如果原生的对象创建可以使用，返回该方法根据原型创建的对象
        if (!_.isObject(prototype)) return {};
        if (nativeCreate) return nativeCreate(prototype);

        // 然后再来处理没有原生对象创建的情况
        // 先将空函数的原型指向要使用的原型，然后创建一个实例
        // 然后再将 Ctor 的原型置为空以供下次使用
        // 最后返回该实例
        // 简单来说就是，如果是 Object.create(null) 的话，就会生成一个没有 prototype 的对象
        // 该对象将不会具有任何基本的 Object 属性（类似 toString() 等）
        Ctor.prototype = prototype;
        var result = new Ctor;
        Ctor.prototype = null;
        return result;
    };

    var property = function (key) {
        return function (obj) {
            return obj == null ? void 0 : obj[key];
        };
    };

    var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;
    var getLength = property('length');
    var isArrayLike = function (collection) {
        var length = getLength(collection);
        return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
    };


    // -----------------------------------------------------------

    // -----------------------------------------------------------
    
    // 中间这一大部分都是定义的一些 ```_.xxx``` 之类的方法

    // -----------------------------------------------------------
    
    // -----------------------------------------------------------
    



    var result = function (instance, obj) {
        return instance._chain ? _(obj).chain() : obj;
    };

    _.mixin = function (obj) {
        _.each(_.functions(obj), function (name) {
            var func = _[name] = obj[name];
            _.prototype[name] = function () {
                var args = [this._wrapped];
                push.apply(args, arguments);
                return result(this, func.apply(_, args));
            };
        });
    };

    _.mixin(_);

    _.each(['pop', 'push', 'reverse', 'shift', 'sort', 'splice', 'unshift'], function (name) {
        var method = ArrayProto[name];
        _.prototype[name] = function () {
            var obj = this._wrapped;
            method.apply(obj, arguments);
            if ((name === 'shift' || name === 'splice') && obj.length === 0) delete obj[0];
            return result(this, obj);
        };
    });

    _.each(['concat', 'join', 'slice'], function (name) {
        var method = ArrayProto[name];
        _.prototype[name] = function () {
            return result(this, method.apply(this._wrapped, arguments));
        };
    });

    _.prototype.value = function () {
        return this._wrapped;
    };

    _.prototype.valueOf = _.prototype.toJSON = _.prototype.value;

    _.prototype.toString = function () {
        return '' + this._wrapped;
    };


    if (typeof define === 'function' && define.amd) {
        define('underscore', [], function () {
            return _;
        });
    }

}.call(this));
