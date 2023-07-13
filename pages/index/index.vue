<template>
	<view class="content">
		<view>长按开始排序，点击事件。
		数据的id如果是字符型，mxj-draggableTreeItem的menuPid数据类型也要对应调整。
		节点的数据的cannotDrag 表示不能移动，  cannotDrop 表示不能拖入。
		</view>
		<mxj-draggableTree @sortFinished="sortFinished" v-model="lists" :max-deep="3" :item-height="40"
			:item-padding-left="20" shadow-color="#eee">
			<view slot-scope={row,$index} class="menu-row" @tap="sortFinished2">
				<view :class="row.cannotDrop?'blue':''">{{row.name}}</view>
			</view>
		</mxj-draggableTree>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				lists: [{
						id: 1,
						name: 'list1',
						children: [{
								id: 2,
								name: 'list2',
								content: {},
								children: [{
									id: 3,
									name: "list3"
								}]
							},
							{
								id: 4,
								name: 'list4',
							}
						]
					},
					{
						id: 5,
						name: 'list5 isFix',
						cannotMove:true,
						cannotDrop:true,
					},
					{
						id: 6,
						name: 'list6',
					},
					{
						id: 7,
						name: 'list7',
					},
				]
			}
		},
		onLoad() {

		},
		methods: {
			sortFinished: function(id, pid, list) {
				//list 是当前节点的兄弟节点列表，用来判断排序
				console.log('sortFinished',id,pid,list)
			},
			sortFinished2: function(id, pid, list) {
				console.log('tap',id,pid,list)
			}
		}
	}
</script>

<style>
	.content {
		padding: 10px;
	}

	.menu-row {
		display: flex;
		align-items: center;
		border-bottom: 1px solid #eee;
		height: 100%;
		background-color:#fff;
	}
	
	.blue{
		color:blue
	}
</style>
