<template>
  <app-layout id="mallOrder">

    <router-link v-if="!isziti" class="mt1 part addr arrow-link" :to="'/address?id='+addr.id||''">
        <div></div>
        <h1>{{addr.id?(addr.consignee+'&nbsp;&nbsp;&nbsp;&nbsp;'+addr.mobilePhone):'请填写收货地址'}}</h1>
        <p>{{addr.id?(addr.proviceName+addr.cityName+addr.areaName+addr.addressDetail):''}}</p>
    </router-link>

    <router-link class="mt1 part goods arrow-link" to="/mall/buyList">
      <img v-if="index<4" v-for="(item,index) in goods" :src="item.img">
      <div>共{{num}}件</div>   
    </router-link>
    <div class="part" style="margin:.05rem 0;" v-if="isziti">
      <h1>提货联系方式</h1>
      <input type="tel" placeholder="请输入提货人联系方式" v-model="tel" style="text-align:right">
    </div>
    <div class="part">
      <h1>配送方式</h1>
      <div>{{shipping}}</div>
    </div>

    <div class="coupon-popup" >
        <div class="arrow-link switch" @click="showCoupon=true">
            <h1>优惠券</h1>
            <div><span :class="{red:coupon.len}">{{coupon.desc||''}}</span></div>
        </div>
    </div>
    
    <use-coupon slot="aside" @close="showCoupon=false" :switch="showCoupon" :goods="goods" v-model="coupon"></use-coupon>

    <div class="mt1 part">
      <p>商品总额：<span class="red">￥{{totle}}</span></p>
      <p v-if="shipping==='快递配送'">运费：<span>￥{{freight}}</span></p>
      <p >优惠券：<span>-￥{{coupon.coupon?coupon.coupon.deductMoney:'0.00'}}</span></p>
    </div>

    <div class="mt1 part">
      <h1>买家留言:</h1>
      <input type="text" placeholder="备注订单说明" v-model="remark">
    </div>

    <div id="footer" slot="footer">
      总计：<span>￥{{final}}</span>
      <button class="next" @click="next">提交订单</button>
    </div>

  </app-layout>
</template>

<script>
import UseCoupon from 'components/coupon/use'
export default {
  components:{
      'use-coupon':UseCoupon,
  },
  data() {
      return {
        addr:{},

        goods:[],
        totle:0,
        curent:0,
        final:0,
        freight:0,
        num:0,
        sid:'',
        shipping:'快递配送',
        fastbuy:true,
        isziti:true,

        showCoupon:false,
        coupon:{},

        remark:'',
        tel:this.$cache.user.phone,

      }
  },
  watch:{
    coupon(data){
      let coupon=data.coupon;

      if(coupon&&coupon.couponId){
        let final=parseFloat(this.$np.minus(this.curent,coupon.deductMoney)).toFixed(2);
        this.final=final>0?final:0;
      }else{
        this.final=this.curent;
      }
      
    }
  },
  mounted(){
    eventBus.$on('selectAddr',(data)=>{
      this.addr=data;
      console.log(this)
      this.getPostfee()
    })
  },
  activated(){
    if(this.goods.length==0){
      this.getData();
    }
  },
  beforeRouteLeave(to, from, next){
    if(/(buyList|address)/.test(to.path)){
      
    }else{
      this.resetPage();
    }
    next();
  },
  methods:{
    resetPage(){
        this.goods=[];
        this.totle=0;
        this.curent=0;
        this.final=0;
        this.freight=0;
        this.num=0;
        this.sid='';
        this.shipping='快递配送';
        this.fastbuy=true,
        this.isziti=true,

        this.coupon={};

        this.remark='';
        
    },
    async getAddress(){
     
        let user=this.$cache.user,
          data={
            content:{
              mbeid:user.memberId,
              mobile:user.phone
            }
        }
        let res= await this.$post('/mall29/addresslist.html', data);
        if (res.errcode === 0) {
 
          const list=res.content.list;
          const arr=list.filter((item)=>{
            return item.isDefault==1
          })

          this.addr=arr[0]||[]
          if(this.addr.id){
            this.getPostfee()
          }
          

        }
    },
    async next(){
      if(!this.fastbuy&&!this.addr.id){
        this.$toast('请填写收货地址');
      }else if(this.isziti&&this.tel==''){
         this.$toast('请输入提货联系方式');
      }else if(this.isziti&&!/^1[3456789]\d{9}$/.test(this.tel)){
        this.$toast('手机号格式有误，请检查');
      }else{
        let goods;
        if(this.$route.query.sid){
          let item=this.goods[0];
          goods=[{itemid:item.item_id,skuid:item.sku_id,num:item.quantity,pType:item.miaosha?'miaosha':'putong'}];
        }else{
          goods=this.goods.map(function(item){
              return {itemid:item.item_id,skuid:item.sku_id,cartid:item.cart_id,num:item.quantity,pType:item.miaosha?'miaosha':'putong'}
          });
        }

        let coupon=this.coupon.coupon
        
        let  data=this.$sign({
              content:{
                orderType:'CP',
                items:goods,
                coupon_code:coupon?coupon.myCouponId:'',
                trade_memo:this.remark,
                totalfee:this.curent,
                address_id:this.addr.id,
                ziti_id:this.$route.query.zid,
                receiver_mobile:this.tel
              }
          });
        let res= await this.$post('/CRM/api/commit.order.json', data);
        if (res.errcode === 0) {
          this.$cache.history.pop();

          if(this.final){
            this.$router.replace('/pay/'+res.content.no);
            eventBus.$emit('reloadDetail');
          }else{
            this.$router.replace('/paid/'+res.content.no);
          }

        }
        
      }
    },
    async getData(){
      let user=this.$cache.user,
          data={
            content:{
              mbeid:user.memberId,
              mobile:user.phone,
              ziti_id:this.$route.query.zid
            }
        };

      let sid=this.$route.query.sid
      if(sid){
        this.sid=sid;
        data.content.sku_id=sid;
        data.content.num=this.$route.query.num;
      }else{
        data.content.cartitems=this.$cache.cart.goods;
      }
      
      let res= await this.$post('/mall29/cart-checkout.html', data);
      if (res.errcode === 0) {

        let con=res.content,
            freight=con.post_fee,
            goods=con.items;

        

        this.fastbuy=con.fastbuy;
        this.isziti=con.isziti;
        this.goods=goods;

        let gi,totle=0,num=0,serviceNum=0;

        for (let i = 0; i < goods.length; i++) {
          gi = goods[i];
          totle=gi.miaosha?this.$np.plus(totle,this.$np.times(gi.miaosha.price,gi.quantity)):this.$np.plus(totle,this.$np.times(gi.price,gi.quantity));
          num+=gi.quantity;
          if(gi.servicetype&&gi.servicetype!=''){
            serviceNum+=gi.quantity
          }
        }

        this.freight=freight;
        this.totle=parseFloat(totle).toFixed(2);
        this.curent=parseFloat(this.$np.plus(totle,freight)).toFixed(2);
        this.final=this.curent;
        this.num=num;
        
        this.$cache.cart.num=num;
        this.$cache.cart.serviceNum=serviceNum;
        this.$cache.cart.totle=totle;
        this.$cache.cart.origin=sid?goods[0]:goods;
        this.$cache.cart.ziti=con.ziti||null;
        if(con.sendtype!=='express'){
          this.shipping='到店服务';
        }else{
          this.getAddress();
        }
        
      }
    },
    async getPostfee(){
   
      let user=this.$cache.user
      let data;
      if(this.sid){
        data={
          content:{
            mbeid:user.memberId,
            num:this.num,
            ziti_id:this.$route.query.zid,
            sku_id:this.sid,
            regionid:this.addr.proviceCode+','+this.addr.cityCode+','+this.addr.areaCode,
            sendtype:'express',
            mobile:user.phone
          }
        }
      }else{
        data={
          content:{
            mbeid:user.memberId,
            sendtype:'express',
            mobile:user.phone,
            cartitems:this.$cache.cart.goods,
            regionid:this.addr.proviceCode+','+this.addr.cityCode+','+this.addr.areaCode,
          }
        }
      }
      

      let res= await this.$post('/mall29/getpostfee.html', data);
      if (res.errcode === 0) {
        this.freight = res.content.post_fee;
        this.curent=parseFloat(this.$np.plus(this.totle,this.freight)).toFixed(2);
        this.final=this.curent;
        // this.postfee = res.content.post_fee>0?'￥'+res.content.post_fee:'包邮'
      }
    },
  }
}
</script>

<style lang="scss">
@import "~assets/css/var.scss";
#mallOrder{
  .mt1{margin-top:.05rem;}
  .red{color:$red;}
  .part{display:block;padding:.1rem .2rem;background: #fff;overflow: hidden;zoom:1;min-height:.24rem;line-height:.24rem;border-bottom:$border;
    h1{color:$black;display: inline-block;}
    div{float:right;color:$gray;margin-right:.1rem;}
    p{color:$gray;font-size:$h3;position: relative;
      span{position:absolute;left:.75rem;top:0;}
    }
    input{width:2.4rem;}
  }

  .addr{padding:.1rem .3rem .1rem .5rem;position:relative;
    div{width:.5rem;position:absolute;height:100%;left:0;top:0;background: url('~assets/img/icon-addr-new.png') no-repeat center;background-size:.15rem .19rem;}
  }
  .goods{text-align: right;line-height:.6rem;
    img{display:block;float:left;width:.6rem;height:.6rem;margin-right:.05rem;}}

  #footer{height:.5rem;line-height: .5rem;background:#fff;padding-left:.2rem;font-size:$h2;border-top:$border;
    span{color:$red;}
    button{float:right;
      &.next{background:$red;height: 100%;width:1.1rem;color:#fff;}
      &.del{color:$red;border:1px solid $red;border-radius: 3px;height:.3rem;margin-top:.1rem;margin-right:.2rem;padding:0 .2rem;line-height: .3rem;}
    }
  }

}
</style>
