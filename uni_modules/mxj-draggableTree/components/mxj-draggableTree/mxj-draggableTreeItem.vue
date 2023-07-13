<template>
	<view>
		<view v-for="(item,i) in list" :class="className + ' ' + 'menu-node'" 
			:style="{'padding-left':itemPaddingLeft + 'px'}"
			:key="item.id"
			:data-menuid="item.id" :data-menupid="menuPid" :data-deep="item._deep" :data-level="item._level" :data-index="i"
			:data-item2="item"
			 v-bind:item1="item"
			@touchstart.stop="menutree.touchstart"
			@touchend="menutree.touchend"
			@touchmove="menutree.touchmove"
			@moveover="menutree.changeNode">
			<view :style="{display:'flex','flex-direction':'column','justify-content':'center','height':itemHeight+'px'}">
				<slot :row="item" :$index="i"></slot>
			</view>
			<mxj-draggableTreeItem v-if="item.children&&item.children.length>0" :list="item.children" :menuPid="item.id" :itemHeight="itemHeight"
			>
				<slot slot-scope="{row,$index}" :row="row" :$index="$index"></slot>
			</mxj-draggableTreeItem>
		</view>
	</view>
</template>
<script module="menutree" lang="renderjs">
	import {
		mxjBus
	} from "./bus.js"
	export default {
		data() {
			return {
				timer:null,
			};
		},
		methods: {
			touchstart: function(e) {
				//console.log(window.event)
				//console.log(e)
				//console.log(e.instance.$el)
				//console.log(e.target.dataset.item)
				//e.$el = window.event.target;
				//console.log(window.event)
				console.log('touchstart',e)
				//console.log(this.list)
				//this.list.push({name:"mxjlinux"})
			console.log('touchstart e.target.item1',e.target.item1)
			console.log('touchstart e.currentTarget.dataset.item2',e.currentTarget.dataset.item2)
			var item=e.currentTarget.dataset.item2;
			if(item.cannotMove)
				return;
				mxjBus.$emit('mxj-draggable-tree-dragstart', e);
			},
			// touchstart: function(item) {
			// 	//console.log(window.event)
			// 	//console.log(e)
			// 	//console.log(e.instance.$el)
			// 	//console.log(e.target.dataset.item)
			// 	//e.$el = window.event.target;
			// 	//console.log(window.event)
			// 	console.log('touchstart',item)
			// 	//console.log(this.list)
			// 	//this.list.push({name:"mxjlinux"})
			
			// 	mxjBus.$emit('mxj-draggable-tree-dragstart', e);
			// },
			touchend:function(e) {
				
				console.log('touchend e.currentTarget.dataset.item2',e.currentTarget.dataset.item2)
				mxjBus.$emit('mxj-draggable-tree-dragend',e);
			},
			touchmove: function(e) {
			
				console.log('touchmove e.currentTarget.dataset.item2',e.currentTarget.dataset.item2)
				// var item=e.currentTarget.dataset.item2;
				// if(item.cannotDrop)
				// 	return;
				if(e.preventDefault) {
					e.preventDefault();
				}
				mxjBus.$emit('mxj-draggable-tree-dragmove',e)
			},
		},
	}
</script>
<script>
	export default {
		name: "mxj-draggableTreeItem",
		props: {
			list: Array,
			className: {
				type: String,
				default: "menu-item"
			},
			itemHeight: {
				type:Number,
				default:40
			},
			itemPaddingLeft: {
				type:Number,
				default: 20
			},
			// menuPid: {
			// 	type:String,
			// 	default:'0'
			// }
			menuPid: {
				type:Number,
				default: 0
			}
		}
	}
</script>

<style lang="scss">
	.menu-item {
		font-size: 1em;
	}
/*未使用*/
	.menu-child {
		font-size: 0.9em;
		font-weight: normal;
	}
</style>
