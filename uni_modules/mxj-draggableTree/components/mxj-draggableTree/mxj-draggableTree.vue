<template>
	<view>
		<mxj-draggableTreeItem ref='menuRootA' v-show="listSwitch" :list="menuList['A']" :itemHeight="itemHeight"
			:itemPaddingLeft="itemPaddingLeft">
			<slot slot-scope="{row,$index}" :row="row" :$index="$index"></slot>
		</mxj-draggableTreeItem>
		<mxj-draggableTreeItem ref='menuRootB' v-show="!listSwitch" :list="menuList['B']" :itemHeight="itemHeight"
			:itemPaddingLeft="itemPaddingLeft">
			<slot slot-scope="{row,$index}" :row="row" :$index="$index"></slot>
		</mxj-draggableTreeItem>
		<view ref="floatMenu" style="position: absolute;z-index:100000;display:none;">
			<view></view>
		</view>
		<view class="menu-item" ref="shadow" :style="{display:'none','padding-left':itemPaddingLeft + 'px'}">
			<view :style="{'background-color':shadowColor,height:'100%'}"></view>
		</view>
		<view style="display:none" ref="params" :data-itemheight="itemHeight" :data-listswitch="listSwitch"
			:data-itempaddingleft="itemPaddingLeft" :data-maxdeep="maxDeep">用于逻辑层向视图层传递数据</view>
	</view>
</template>
<script>
	import {
		bu
	} from "./bus.js"
	import mxjDraggableTreeItem from "./mxj-draggableTreeItem.vue"
	export default {
		name: "mxj-draggableTree",
		components: {
			mxjDraggableTreeItem
		},
		props: {
			value: Array,
			itemHeight: {
				type: Number,
				default: 40
			},
			itemPaddingLeft: {
				type: Number,
				default: 20
			},
			maxDeep: {
				type: Number,
				default: 3
			},
			shadowColor: {
				type: String,
				default: '#eee'
			}
		},
		data() {
			return {
				menuList: {
					'A': [],
					'B': []
				},
				listSwitch: true,
			};
		},
		created() {
			this.initList(this.value);
		},
		mounted() {},
		watch: {
			value: function() {
				this.switchList(this.value);
				//this.$forceUpdate()
			}
		},
		methods: {
			initList: function(list) {
				list = JSON.parse(JSON.stringify(list));
				this.setDeepAndLevel(list);
				this.menuList['A'] = list;
			},
			switchList: function(list) {
				list = JSON.parse(JSON.stringify(list));
				this.setDeepAndLevel(list);
				let listType = this.listSwitch ? "B" : "A";
				//先清空，避免dom与vdom不一致造成渲染错误
				this.menuList[listType] = [];
				this.$nextTick(function() {
					this.menuList[listType] = list;
				})

				let that = this;
				//延时50mm切换显示，可能会避免奇怪的错误，待验证
				setTimeout(() => {
					that.listSwitch = !that.listSwitch;
				}, 50)
			},
			setDeepAndLevel: function(list, level = 0) {
				let deeps = [];
				for (let el of list) {
					el._level = level;
					if (el.children && el.children.length > 0) {
						el._deep = 1 + this.setDeepAndLevel(el.children, level + 1);
					} else {
						el._deep = 0;
					}
					deeps.push(el._deep);
				}
				return Math.max(...deeps)
			},
			menuFlat: function(list) {
				return list.reduce((res, item) => {
					res.push(item);
					if (item.children && item.children.length > 0) {
						return res.concat(this.menuFlat(item.children))
					}
					return res;
				}, []);
			},
			sortFinished: function({
				id,
				pid,
				oldpid,
				list
			}) {
				let newList = this.updateList(id, pid, oldpid, list);
				this.$emit("input", newList);
				this.$emit("sortFinished", id, pid, list)
			},
			updateList: function(id, pid, oldpid, list) {
				let newList = JSON.parse(JSON.stringify(this.value));
				let flatList = this.menuFlat(newList);
				let newIndex = list.indexOf(id);
				let item = this.getListItem(id, flatList);

				if (oldpid.toString() === '0') {
					newList.splice(newList.indexOf(item), 1);
				} else {
					let oldParent = this.getListItem(oldpid, flatList);
					oldParent.children.splice(oldParent.children.indexOf(item), 1);
				}
				if (pid.toString() == '0') {
					newList.splice(newIndex, 0, item)
				} else {
					let parent = this.getListItem(pid, flatList);
					if (parent.children) {
						parent.children.splice(newIndex, 0, item);
					} else {
						parent.children = [item];
					}
				}
				return newList;

			},
			getListItem: function(id, list) {
				for (let el of list) {
					if (el.id.toString() == id.toString()) {
						return el;
					}
				}
				return false;
			},
			vibrate: function() {
				uni.vibrateShort()
			}
		},
	}
</script>
<script module="menuroot" lang="renderjs">
	import {
		mxjBus
	} from "./bus.js"
	export default {
		data() {
			return {
				timer: null,
				moving: false,
				moved: false,
				menuRoot: null,
				//HtmlCollection,缓存menu doms,排序、添加、移除等操作，menuNodes自动更新，无需干预
				menuNodes: null,
				//当前移动元素
				currentMoveNode: null,
				//根节点位置
				menuRootPosition: null,
				//跟随节点移动的元素
				floatMenu: null,
				//被移动的菜单再列表中显示为阴影，跟随floatMenu
				shadow: null,
				//当前触摸位置
				touchPoint: null,
				//floatMenu初始位置,用于计算shadow跟随触摸位置
				floatMenuInitPoint: null,
				//被移动元素位置,用于计算超出临界执行元素置换
				srcPoint: null,
				params: {}
			}
		},
		created() {
			mxjBus.$on('mxj-draggable-tree-dragstart', (e) => {
				//this.$ownerInstance.callMethod("vibrate")
				let that = this;
				this.timer = setTimeout(function() {
					that.moving = true;
					that.dragStart(e);
				}, 500);

			})
			mxjBus.$on('mxj-draggable-tree-dragmove', (e) => {
				if (this.moving) {
					this.dragMove(e);
				} else {
					clearTimeout(this.timer);
				}
			})
			mxjBus.$on('mxj-draggable-tree-dragend', (e) => {
				clearTimeout(this.timer);
				if (this.moving) {
					this.dragEnd(e);
					this.moving = false;
				}
			})
		},
		mounted() {
			this.floatMenu = this.$refs.floatMenu;
			this.shadow = this.$refs.shadow;
		},
		methods: {
			dragStart: function(e) {
				this.touchPoint = e.touches[0];
				this.srcPoint = {
					clientX: e.currentTarget.offsetLeft,
					clientY: e.currentTarget.offsetTop,
					clientHeight: e.instance.$el.clientHeight,

					//用于恢复dom原始结构(直接操作dom，导致vdom与dom不一致，vue数据更新后渲染错误，因此更新数据前需恢复dom)
					parentNode: e.instance.$el.parentNode,
					//只有一个子菜单时，移动子菜单，父菜单view会被移除，此情况需要记录爷爷级，此时父级为爷爷级的最后一项，恢复时直接append
					pparentNode: e.instance.$el.parentNode.parentNode,
					//记录在父级中的索引
					index: parseInt(e.instance.$el.dataset.index)
				};
				this.srcPoint.nextNode = this.srcPoint.parentNode.children[this.srcPoint.index + 1] ? this.srcPoint
					.parentNode.children[this.srcPoint.index + 1] : null;

				this.floatMenuInitPoint = this.srcPoint;
				this.currentMoveNode = e.instance.$el;
				this.initParams();
				this.initMenuRoot();
				this.showFloatMenu();
				this.moving = true;
			},
			dragMove: function(e) {
				let x = this.touchPoint.clientX - e.touches[0].clientX;
				let y = this.touchPoint.clientY - e.touches[0].clientY;
				let parentNode = this.shadow.$el.parentNode;
				this.moveFloatMenu(x, y);
				//移除空的<uni-view><uni-view>
				if (parentNode.children.length == 0) {
					parentNode.parentNode.removeChild(parentNode);
				}
			},
			findItemById(id,list){
				for(var i of list){
					if(i.id==id){
						return i;
					}
					if(i.children&&i.children.length>0){
						var item=this.findItemById(id,i.children);
						if(item){
							return item;
						}
					}
				}
			},
			dragEnd: function(e) {
				this.hideFloatMenu();
				//判断是否发生排序
				// this.initList(this.value);
				let pid = this.currentMoveNode.parentNode == this.menuRoot.$el ? 0 : this.currentMoveNode.parentNode
					.parentNode.dataset
					.menuid;
					console.log(this.currentMoveNode.parentNode.dataset )
				// console.log('moveEnd',JSON.stringfy( this.currentMoveNode.parentNode.dataset ))
				console.log('moveEnd item2', this.currentMoveNode.parentNode.dataset.item2 )
				console.log('moveEnd item1', this.currentMoveNode.parentNode.dataset.item1 )
				console.log('moveEnd pid', pid )
				var item=this.findItemById(pid,this.value);
				console.log('drop item', item )
				if(item&&item.cannotDrop){
					this.switchList(this.value);
					return;
				}
				
				if (!(this.currentMoveNode.offsetTop == this.srcPoint.clientY && this.currentMoveNode.offsetLeft ==
						this.srcPoint
						.clientX)) {
					this.moveEnd();
					setTimeout(this.recoveryDom, 100)
				}

			},
			initMenuRoot: function() {
				this.menuRoot = this.$refs['menuRoot' + (this.params.listSwitch ? 'A' : 'B')];
				this.menuRootPosition = {
					offsetTop: this.menuRoot.$el.offsetTop,
					offsetLeft: this.menuRoot.$el.offsetLeft,
					offsetHeight: this.menuRoot.$el.offsetHeight,
					offsetWidth: this.menuRoot.$el.offsetWidth
				};
				this.menuNodes = this.menuRoot.$el.getElementsByClassName('menu-node');
			},
			initParams: function() {
				this.params.itemHeight = parseInt(this.$refs.params.$el.dataset.itemheight);
				this.params.itemPaddingLeft = parseInt(this.$refs.params.$el.dataset.itempaddingleft);
				this.params.maxDeep = parseInt(this.$refs.params.$el.dataset.maxdeep);
				this.params.listSwitch = (String(this.$refs.params.$el.dataset.listswitch) == 'true');
			},
			showFloatMenu: function() {
				let floatMenu = this.floatMenu.$el;
				let shadow = this.shadow.$el;
				floatMenu.style.width = this.currentMoveNode.offsetWidth + 'px';
				floatMenu.replaceChild(this.currentMoveNode.cloneNode(true), floatMenu.childNodes[0]);
				floatMenu.style.top = this.currentMoveNode.offsetTop - 2 + 'px';
				floatMenu.style.left = this.currentMoveNode.offsetLeft + 2 + 'px';
				floatMenu.style.display = "block";
				shadow.style.height = this.currentMoveNode.offsetHeight + 'px';
				shadow.style.display = "block";
				this.currentMoveNode.parentNode.replaceChild(shadow, this.currentMoveNode);

			},
			hideFloatMenu: function() {
				this.floatMenu.$el.style.display = "none"
				let shadow = this.shadow.$el;
				shadow.parentNode.replaceChild(this.currentMoveNode, shadow);
			},
			recoveryDom: function() {
				let parentNode = this.srcPoint.parentNode;
				let nextNode = this.srcPoint.nextNode;
				let oldParent = this.currentMoveNode.parentNode;
				if (nextNode) {
					this.srcPoint.parentNode.insertBefore(this.currentMoveNode, nextNode);
				} else if (parentNode.children.length == 0) {
					parentNode.appendChild(this.currentMoveNode);
					this.srcPoint.pparentNode.appendChild(parentNode);
				} else {
					parentNode.appendChild(this.currentMoveNode)
				}
				if (oldParent.children.length == 0) {
					oldParent.parentNode.removeChild(oldParent);
				}
			},
			moveFloatMenu: function(x, y) {
				let itemHeight = this.params.itemHeight;
				let floatMenu = this.floatMenu.$el;
				let shadow = this.shadow.$el;
				let targetX = this.floatMenuInitPoint.clientX - x;
				let targetY = this.floatMenuInitPoint.clientY - y;
				/*限制移动范围
				if (targetY > this.menuRootPosition.offsetTop + this.menuRootPosition.offsetHeight - shadow
					.offsetHeight) {
					targetY = this.menuRootPosition.offsetTop + this.menuRootPosition.offsetHeight - shadow
						.offsetHeight;
				} else if (targetY < this.menuRootPosition.offsetTop) {
					targetY = this.menuRootPosition.offsetTop;
				}
				
								if (targetX > shadow.offsetLeft + this.itemPaddingLeft) {
									targetX = shadow.offsetLeft + this.itemPaddingLeft;
								} else if (targetX < shadow.offsetLeft - this.itemPaddingLeft) {
									targetX = shadow.offsetLeft - this.itemPaddingLeft;
								}
				*/
				floatMenu.style.top = targetY + 'px';
				floatMenu.style.left = targetX + 'px';
				let node;
				//向下移动,index下标从0开始
				if ((targetY - shadow.offsetTop) > itemHeight * 0.6) {
					node = this.getNode(parseInt((targetY - this.menuRootPosition.offsetTop) / itemHeight));
					this.moveShadowAfter(node);
				}
				//向上移动
				else if (((shadow.offsetTop - targetY) > itemHeight * 0.6) && (targetY > this.menuRootPosition
						.offsetTop)) {
					node = this.getNode(parseInt((targetY - this.menuRootPosition.offsetTop) / itemHeight));
					this.moveShadowBefore(node);
				}
				//左移
				if ((shadow.offsetLeft - targetX) > (this.params.itemPaddingLeft * 0.7)) {
					this.moveShadowLeft();
				}
				//右移
				else if ((targetX - shadow.offsetLeft) > (this.params.itemPaddingLeft * 0.7)) {
					this.moveShadowRight();
				}
			},
			moveShadowAfter: function(node) {
				let nodeChilds = node.getElementsByClassName("menu-node");
				if (nodeChilds.length > 0) {
					if ((parseInt(this.currentMoveNode.dataset.deep) + parseInt(node.dataset.level) + 1) < this.params
						.maxDeep) {
						nodeChilds[0].parentNode.insertBefore(this.shadow.$el, nodeChilds[0]);
						this.floatMenu.$el.width = nodeChilds[0].offsetWidth + 'px';
					}
				} else {
					let parent = node.parentNode;
					if (!parent.parentNode.dataset.level || (parseInt(this.currentMoveNode.dataset.deep) + parseInt(
							node.dataset
							.level)) < this.params.maxDeep) {
						if (this.getLastNode(node) == node) {
							parent.appendChild(this.shadow.$el);
						} else {
							parent.insertBefore(this.shadow.$el, this.getNextNode(node))
						}

						this.floatMenu.$el.width = node.offsetWidth + 'px';
					}
					//跨子节点移动（超出了最大深度）,当前节点为最后一个子节点且无子节点
					else if (this.getLastNode(node) == node) {
						while (parent.parentNode.dataset.level && (parseInt(this.currentMoveNode.dataset.deep) +
								parseInt(parent.parentNode.dataset.level)) >= this.params.maxDeep) {
							parent = parent.parentNode.parentNode;
							//判断每一级是否为本级最后一个节点
							if (this.getLastNode(parent.parentNode) != parent.parentNode) {
								parent = false;
								break;
							}
						}
						if (parent) {
							parent = parent.parentNode;
							if (this.getLastNode(parent) == parent) {
								parent.parentNode.appendChild(this.shadow.$el);
							} else {
								parent.parentNode.insertBefore(this.shadow.$el, this.getNextNode(parent))
							}

							this.floatMenu.$el.width = parent.offsetWidth + 'px';
						}
					}
				}

			},
			moveShadowBefore: function(node) {
				let parent = node.parentNode;
				if (!parent.parentNode.dataset.level || (parseInt(this.currentMoveNode.dataset.deep) + parseInt(node
						.dataset
						.level)) < this.params.maxDeep) {
					parent.insertBefore(this.shadow.$el, node);
					this.floatMenu.$el.width = node.offsetWidth + 'px';
				}
			},
			moveShadowLeft: function() {
				let shadow = this.shadow.$el;
				if (this.getLastNode(shadow) == shadow) {
					if (shadow.parentNode != this.menuRoot.$el) {
						//let node = shadow.parentNode.nextSibling;

						//shadow.parentNode.parentNode.insertBefore(shadow, node);

						let node = shadow.parentNode.parentNode;
						if (this.getLastNode(node) == node) {
							node.parentNode.appendChild(shadow);
						} else {
							node.parentNode.insertBefore(shadow, this.getNextNode(node));
						}
					}
				}
			},
			moveShadowRight: function() {
				let shadow = this.shadow.$el;
				let node;
				if (node = this.getPreviousNode(shadow)) {
					if ((parseInt(node.dataset.level) + parseInt(this.currentMoveNode.dataset.deep) + 1) < this.params
						.maxDeep) {
						//子菜单与父菜单之间有个view，children[0]为本级菜单显示，children[1]子菜单view
						if (!node.children[1]) {
							//如无子菜单view，创建
							node.appendChild(document.createElement('uni-view'));
						}
						node.children[1].appendChild(shadow);
					}
				}
			},
			moveEnd: function() {
				let id = this.currentMoveNode.dataset.menuid;
				let pid = this.currentMoveNode.parentNode == this.menuRoot.$el ? 0 : this.currentMoveNode.parentNode
					.parentNode.dataset
					.menuid;
				
				let oldpid = this.currentMoveNode.dataset.menupid;
				let list = [];
				for (let el of this.currentMoveNode.parentNode.children) {
					list.push(el.dataset.menuid);
				}
				this.$ownerInstance.callMethod("sortFinished", {
					id: id,
					pid: pid,
					oldpid: oldpid,
					list: list
				})
			},
			getNode: function(i) {
				if (this.menuNodes.length === 0) {
					this.menuNodes = this.menuRoot.$el.getElementsByClassName('menu-node');
				}
				i = i >= this.menuNodes.length - 1 ? this.menuNodes.length - 1 : i;
				return this.menuNodes[i] ? this.menuNodes[i] : null;
			},
			getLastNode: function(node) {
				let children = node.parentNode.children;
				return children[children.length - 1];
			},
			getNextNode: function(node) {
				let children = Array.prototype.slice.call(node.parentNode.children);
				let index = children.indexOf(node);
				return children[index + 1] ? children[index + 1] : false;
			},
			getPreviousNode: function(node) {
				let children = Array.prototype.slice.call(node.parentNode.children);
				let index = children.indexOf(node);
				return children[index - 1] ? children[index - 1] : false;
			}
		},
		beforeDestroy() {
			mxjBus.$off("mxj-draggable-tree-dragstart");
			mxjBus.$off("mxj-draggable-tree-dragmove");
			mxjBus.$off("mxj-draggable-tree-dragend");
		}
	}
</script>


<style lang="scss">

</style>
