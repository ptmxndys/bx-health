<template>
	<view class="article-list">
<!-- 		<view class="action margin-top-xs">
			<text class="cuIcon-titles text-cyan"></text>
			<text class="text-bold">{{ config.item_label }}</text>
		</view> -->
		<u-tabs :list="tabs" :is-scroll="true" activeColor="#0bc99d" ref="tabs" :current="current" @change="changeTab" v-if="tabs.length>1"></u-tabs>
		<view class="">
			<view class="article-item" v-for="item in list" @click="toDetail(item)">
				<view class="title"><text class="cuIcon-text margin-right-xs"></text> {{ item.title }}</view>
				<view class="date">{{ formateDate(item.create_time) }}</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			current: 0,
			tabs: [],
			list: []
		};
	},
	props: {
		config: {
			type: Object
		}
	},
	methods: {
		getTabContent() {
			let req = {
				condition: [{ colName: 'no', ruleType: 'like', value: this.tabs[this.current].no }],
				page: { pageNo: 1, rownumber: 5 }
			};
			this.$fetch('select', 'srvdaq_cms_content_select', req, 'daq').then(res => {
				if (res.success) {
					this.list = res.data;
				}
			});
		},
		toDetail(e){
			uni.navigateTo({
				url:'/publicPages/article/article?content_no='+e.content_no
			})
		},
		changeTab(e) {
			this.current = e;
		}
	},
	created() {
		if (Array.isArray(this.config.tablist) && this.config.tablist.length > 0) {
			this.tabs = this.config.tablist.map(item => {
				item.name = item.tablist_name;
				return item;
			});
			this.getTabContent();
		}
	}
};
</script>

<style scoped lang="scss">
.article-item {
	padding:10px;
	display: flex;
	align-items: center;
	justify-content: space-between;
	border-bottom: 1px dashed #f1f1f1;
	box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12), 0 0 6px rgba(0, 0, 0, 0.04);
	margin-bottom: 10px;
	border-radius: 5px;
	&:active{
		background-color: #f1f1f1;
	}
	.title{
		flex: 1;
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
	}
	.date {
		color: #777;
	}
}
</style>
