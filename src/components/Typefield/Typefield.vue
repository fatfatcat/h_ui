<template>
  <div v-if="!hidden" :class="clazz">
    <input 
      :disabled="disabled" 
      :readonly="!editable||readonly" 
      :placeholder="placeholder" 
      :value="inputValue" 
      @blur="blur" 
      @input="val" 
      @focus="focus($event)" 
      ref="input">
    <transition name="label-fade">
      <div v-show="tipShow" :class="tipzz">{{bigNum}}</div>
    </transition>
  </div>
</template>
<script>
import {oneOf} from '../../util/tools'
import Emitter from '../../mixins/emitter';

  Number.prototype.toFixed = function (n) {
    if (n > 20 || n < 0) {
        throw new RangeError('toFixed() digits argument must be between 0 and 20');
    }
    const number = this;
    if (isNaN(number) || number >= Math.pow(10, 21)) {
        return number.toString();
    }
    if (typeof (n) == 'undefined' || n == 0) {
        return (Math.round(number)).toString();
    }
    let result = number.toString();
    const arr = result.split('.');
    // 整数的情况
    if (arr.length < 2) {
        result += '.';
        for (let i = 0; i < n; i += 1) {
            result += '0';
        }
        return result;
    }
    const integer = arr[0];
    const decimal = arr[1];
    if (decimal.length == n) {
        return result;
    }
    if (decimal.length < n) {
        for (let i = 0; i < n - decimal.length; i += 1) {
            result += '0';
        }
        return result;
    }
    result = integer + '.' + decimal.substr(0, n);
    const last = decimal.substr(n, 1);

    // 四舍五入，转换为整数再处理，避免浮点数精度的损失
    if (parseInt(last, 10) >= 5) {
        const x = Math.pow(10, n);
        result = (Math.round((parseFloat(result) * x)) + 1) / x;
        result = result.toFixed(n);
    }
    return result;
  };
const prefixCls = 'h-typefield';
export default {
  name: 'Typefield',
  data(){
    return {
      focused: false,
      inputValue: '',
      checked:false,
      tipShow:false,
      bigNum:null
    }
  },
  mixins: [ Emitter ],
  props: {
    type: {
      validator (value){
        return oneOf(value,['money','cardNo'])
      },
      default:'money'
    },
    value: {
      required: false
    },
    disabled: {
      type:Boolean,
      default:false
    },
    readonly: {
      type:Boolean,
      default:false
    },
    editable: {
      type:Boolean,
      default:true
    },
    placeholder: String,
    hidden: Boolean,
    isround:Boolean,
    integerNum: {
      type: [Number,String],
      default: 10
    },
    suffixNum:{
      type: [Number,String],
      default: 3
    },
    bigTips:Boolean,
    position:String,
  },
  computed: {
    clazz () {
      return [
        `${prefixCls}`,
        {
          'input-field-focused': this.focused,
          'input-field-active': this.inputValue || this.placeholder || (this.$refs.input && this.$refs.input.value),
          [`${prefixCls}-disabled`]: this.disabled,
          [`${prefixCls}-readonly`]: this.readonly,
          [`${prefixCls}-editable`]: !this.editable
        }
      ]
    },
    tipzz () {
      return `${prefixCls}-tip`
    }
  },
  watch: {
    value (value) {
      this.inputValue = value;
    }
  },
  mounted () {
    this.inputValue = this.value
  },
  methods:{
    //keyup,focus,blur
    blur (e) {
      if (this.type=='money') {
        this.tipShow=false
      }
      if (this.inputValue&&this.inputValue!="") {
        this.inputValue = this.formatNum(e.target.value)
      }
      this.$emit('input', this.inputValue);
      this.$emit('on-blur')
      this.dispatch('FormItem', 'on-form-blur', this.inputValue)
    },
    focus (e) {
      this.focused = true
      this.bigShow(this.type,this.bigTips,this.inputValue)
      this.$emit('on-focus',e)
    },
    val (e) {
      this.inputValue = e.target.value;
      this.bigShow(this.type,this.bigTips,this.inputValue);
      this.$emit('input', this.inputValue);
      this.$emit('on-keyup', this.inputValue);
    },
    changeTipsVal (value){   
      value  = String(value).replace(/[^0-9\.-]/g,"");
      var firstChar = value.substring(0,1); 
      value = this.cutNum(value,this.integerNum);
      if(value.split(".")[1] && value.split(".")[1].length > 2){
        var isround = this.isround;
        if(isround&&isround==true){
          value = parseFloat(value).toFixed(this.suffixNum);
        }else{
          var suf = value.split(".")[1].substr(0,this.suffixNum);
          value = value.split(".")[0]+"."+suf;
        }
      }
      return this.numtochinese(value + "",this.suffixNum);
    },
    formatNum (value){
      var suffixNum = this.suffixNum;
      var integerNum = this.integerNum;  
      value  = value.replace(/[^0-9\.-]/g,"")||'';
      var firstChar = value.substring(0,1)||'';
      if(this.type == "cardNo"){
        value=value.replace("-", "").replace(".", "")
        if(value!=null&&value!=""){
          if(value.indexOf(".")!=-1){
            value = value.split(".")[0];
          }
          value = value? value.match(/\d{1,4}/g).join("  "):"";
        }
      }
      if (this.type == "money") {
        if (firstChar=='-') {
          value = value.substring(1)||'';
        }
        value=value.replace("-", "")
        value = this.cutNum(value,this.integerNum);
        if (value=='') return;
        if (this.isround) {
          value = Number(value).toFixed(suffixNum);
        }else {
          value = this.fillZero(Number(value),suffixNum)
        }
        value = this.setNum(value,suffixNum,integerNum)
        if (firstChar=='-') {
          value = '-'+value;
        }
      }
      return value
    },
    numtochinese (Num,suffixNumber) {
      for (var i = Num.length - 1; i >= 0; i--) {
        Num = Num.replace(",", "")// 替换tomoney()中的“,”
        Num = Num.replace(" ", "")// 替换tomoney()中的空格
      }
      Num = Num.replace("￥", "")// 替换掉可能出现的￥字符
      if (isNaN(Num)) {
        // 验证输入的字符是否为数字
        // alert("请检查小写金额是否正确");
        return;
      }
      // ---字符处理完毕，开始转换，转换采用前后两部分分别转换---//
      var part = String(Num).split(".");
      var newchar = "";
      // 小数点前进行转化
      for (i = part[0].length - 1; i >= 0; i--) {
        if (part[0].length > 17) {
        //   alert("");
          return "位数过大，无法计算";
        }// 若数量超过拾亿单位，提示
        var tmpnewchar = ""
        var perchar = part[0].charAt(i);
        switch (perchar) {
        case "0":
          tmpnewchar = "零" + tmpnewchar;
          break;
        case "1":
          tmpnewchar = "壹" + tmpnewchar;
          break;
        case "2":
          tmpnewchar = "贰" + tmpnewchar;
          break;
        case "3":
          tmpnewchar = "叁" + tmpnewchar;
          break;
        case "4":
          tmpnewchar = "肆" + tmpnewchar;
          break;
        case "5":
          tmpnewchar = "伍" + tmpnewchar;
          break;
        case "6":
          tmpnewchar = "陆" + tmpnewchar;
          break;
        case "7":
          tmpnewchar = "柒" + tmpnewchar;
          break;
        case "8":
          tmpnewchar = "捌" + tmpnewchar;
          break;
        case "9":
          tmpnewchar = "玖" + tmpnewchar;
          break;
        }
        switch (part[0].length - i - 1) {
        case 0:
          tmpnewchar = tmpnewchar + "元";
          break;
        case 1:
          if (perchar != 0)
            tmpnewchar = tmpnewchar + "拾";
          break;
        case 2:
          if (perchar != 0)
            tmpnewchar = tmpnewchar + "佰";
          break;
        case 3:
          if (perchar != 0)
            tmpnewchar = tmpnewchar + "仟";
          break;
        case 4:
          tmpnewchar = tmpnewchar + "万";
          break;
        case 5:
          if (perchar != 0)
            tmpnewchar = tmpnewchar + "拾";
          break;
        case 6:
          if (perchar != 0)
            tmpnewchar = tmpnewchar + "佰";
          break;
        case 7:
          if (perchar != 0)
            tmpnewchar = tmpnewchar + "仟";
          break;
        case 8:
          tmpnewchar = tmpnewchar + "亿";
          break;
        case 9:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "拾";
          break;
        case 10:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "百";
          break;
        case 11:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "仟";
          break;
        case 12:
          tmpnewchar = tmpnewchar + "兆";
          break;
        case 13:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "拾";
          break;
        case 14:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "百";
          break;
        case 15:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "仟";
          break;
        case 16:
          if (perchar != 0)
          tmpnewchar = tmpnewchar + "京";
          break;
        case 17:
          tmpnewchar = tmpnewchar + "拾";
          break;
        }
        newchar = tmpnewchar + newchar;
      }
      // 小数点之后进行转化
      if (Num.indexOf(".") != -1) {
        if (part[1].length > 2) {
          //alert("小数点之后只能保留两位,系统将自动截段");
          var tempNum = parseFloat(Num);
          Num = tempNum.toFixed(suffixNumber);
          part = String(Num).split(".");
        }
        for (i = 0; i < part[1].length; i++) {
          tmpnewchar = ""
          perchar = part[1].charAt(i)
          switch (perchar) {
          case "0":
            tmpnewchar = "零" + tmpnewchar;
            break;
          case "1":
            tmpnewchar = "壹" + tmpnewchar;
            break;
          case "2":
            tmpnewchar = "贰" + tmpnewchar;
            break;
          case "3":
            tmpnewchar = "叁" + tmpnewchar;
            break;
          case "4":
            tmpnewchar = "肆" + tmpnewchar;
            break;
          case "5":
            tmpnewchar = "伍" + tmpnewchar;
            break;
          case "6":
            tmpnewchar = "陆" + tmpnewchar;
            break;
          case "7":
            tmpnewchar = "柒" + tmpnewchar;
            break;
          case "8":
            tmpnewchar = "捌" + tmpnewchar;
            break;
          case "9":
            tmpnewchar = "玖" + tmpnewchar;
            break;
          }
          if (i == 0)
            tmpnewchar = tmpnewchar + "角";
          if (i == 1)
            tmpnewchar = tmpnewchar + "分";
          newchar = newchar + tmpnewchar;
        }
      }
      //替换所有无用汉字
      while (newchar.search("零零") != -1)
        newchar = newchar.replace("零零", "零");
        newchar = newchar.replace("零亿", "亿");
        newchar = newchar.replace("亿万", "亿");
        newchar = newchar.replace("零万", "万");
        newchar = newchar.replace("零元", "元");
        newchar = newchar.replace("零角", "");
        newchar = newchar.replace("零分", "");
              
        newchar = newchar.replace("亿万", "亿");
        newchar = newchar.replace("兆亿", "兆");
        newchar = newchar.replace("零兆", "兆");
        newchar = newchar.replace("京兆", "京");
      
      if (newchar.charAt(newchar.length - 1) == "元"
          || newchar.charAt(newchar.length - 1) == "角")
        newchar = newchar + "整";

      var digit = ['壹', '贰', '叁', '肆', '伍', '陆', '柒', '捌', '玖'];
      var _i = 0
      while(newchar.length > 0){
        if(digit.indexOf(newchar[0]) < 0){
          newchar = newchar.substr(1);
        }else{
          break;
        }
      }

      var firstChar = Num.substring(0,1);
      if(firstChar=="-"){
        newchar = "负"+newchar;
      }
     
      var lastChar=newchar.charAt(newchar.length-1);
      if("零" == lastChar){
        newchar=newchar.substring(0,newchar.length-1);
        newchar=newchar+"整";
      }

      if(parseFloat(Num)==0){
        newchar="零元整";
      }
      return newchar;
    },
    fillZero(number, bitNum) {  
      /// 小数位不够，用0补足位数   
      var f_x = parseFloat(number);  
      if (isNaN(f_x)) {  
        return ;  
      }  
      var s_x = number.toString();  
      var pos_decimal = s_x.indexOf('.');  
      if (pos_decimal < 0) {  
        pos_decimal = s_x.length;  
        s_x += '.';  
      }  
      while (s_x.length <= pos_decimal + bitNum) {  
        s_x += '0';  
      }  
      return s_x;  
    },
    cutNum(value,integerNum){
      var arrNum=value.split(".");
      if(arrNum.length>0){
        var integerNumber=arrNum[0].substr(0,this.integerNum);
        if(arrNum.length>1){
          value=integerNumber+"."+arrNum[1]
        }else{
          value=integerNumber
        }        
      }
      return value;
    },
    setNum(value,suffixNum,integerNum){
      if (isNaN(value)) return ;
      if (suffixNum>0) {
        var arrNum=value.split(".");
        var integerNumber = arrNum[0].substring(0,integerNum);
        value = integerNumber+'.'+arrNum[1].substring(0,suffixNum);
      }
      return value
    },
    bigShow(type,bigTips,value){
      if (type=='money'&&this.bigTips&&value) {
        this.bigNum = this.changeTipsVal(value)
        this.tipShow = Boolean(this.bigNum)
      }
    }  

  }
}
</script>