<template>
	<view class="store-wrap" v-if="!authBoxDisplay"><shop-store ref="store"></shop-store></view>
	<bx-auth v-else-if="authBoxDisplay" @getuserinfo="getuserinfo"></bx-auth>
</template>

<script>
//引入测试数据
import shopStore from '@/components/shopStore/shopStore.vue';
import { mapGetters } from 'vuex';
export default {
	components: { shopStore },
	computed: {
		...mapGetters({
			authSetting: 'authSetting',
			is_login: 'isLogin',
			wxUserInfo: 'wxUserInfo',
			login_user_info: 'loginUserInfo',
			client_env: 'env',
			authBoxDisplay: 'authBoxDisplay'
		})
	},
	onLoad(option) {
		this.checkOptionParams(option);
	},
	data() {
		return {
			isLogin: false, //是否已经登录
			MPPR: 0,
			searchVal: '', //搜索内容
			pageInfo: {
				total: 0,
				rownumber: 8,
				pageNo: 1
			},
			//显示没有更多商户
			showFoot: 0,
			downOption: {
				auto: false //是否在初始化后,自动执行downCallback; 默认true
			},
			sortIndex: 0,
			sortList: [{ tit: '商户', type: 'shop' }, { tit: '我的店铺', type: 'myShop' }],
			myStoreList: [], //当前登录人得商铺列表
			current_tit: { tit: '商户', type: 'shop' }, //当前点击顶部分类
			storeList: [],
			shopList: []
		};
	},
	async mounted() {
		let self = this;
		uni.$on('loginStatusChange', result => {
			self.isLogin = result;
		});
		let userInfo = uni.getStorageSync('login_user_info');
		if (!userInfo) {
			// 未登录， 提示跳转
			self.isLogin = false;
			let res = await uni.showModal({
				title: '提示',
				content: '未登录,是否跳转到登录页面?',
				confirmText: '去登录',
				confirmColor: '#02D199'
			});
			if (Array.isArray(res) && res.length > 1) {
				if (res[1].confirm) {
					// 确认 跳转到登录页
					uni.navigateTo({
						url: '/publicPages/accountExec/accountExec'
					});
				} else if (res[1].cancel) {
					// 取消 返回首页
					uni.switchTab({
						url: '/pages/pedia/pedia'
					});
				}
			}
		} else {
			self.isLogin = true;
		}
	},
	onShow() {
		if (this.isLogin) {
			this.$refs.store.getShopList();
			this.$refs.store.onRefresh();
		}
	},
	methods: {
		async getuserinfo(e) {
			// #ifdef MP-WEIXIN
			const user = e.mp.detail;
			if (user && user.userInfo) {
				let rawData = {
					nickname: user.userInfo.nickName,
					sex: user.userInfo.gender,
					country: user.userInfo.country,
					province: user.userInfo.province,
					city: user.userInfo.city,
					headimgurl: user.userInfo.avatarUrl
				};
				this.setWxUserInfo(rawData);
				this.$store.commit('SET_WX_USERINFO', rawData);
				this.$store.commit('SET_AUTH_SETTING', { type: 'userInfo', value: true });
				this.$store.commit('SET_AUTH_USERINFO', true);
				this.toAddPage();
			}
			// #endif
		},
		onRefresh() {
			this.pageInfo.pageNo = 1;
			this.$nextTick(() => {
				this.$refs.pullScroll.refresh();
			});
		},
		pullDown(pullScroll) {
			console.log(pullScroll.page);
			let page = this.pageInfo;
			this.pageInfo.pageNo = 1;
			let self = this;
			setTimeout(() => {
				this.loadData(pullScroll);
			}, 200);
		},
		loadData(pullScroll) {
			console.log('上拉加载');
			let page = this.pageInfo;
			this.pageInfo.pageNo = pullScroll.page;
			console.log(pullScroll.page);
			if (this.current_tit.type === 'shop') {
				this.getShopList();
			} else if (this.current_tit.type === 'myShop') {
				this.getMyShopList();
			}
		},
		amend(item) {
			let cond = [
				{
					colName: 'id',
					ruleType: 'in',
					value: item.id
				}
			];
			let params = {
				type: 'update',
				condition: cond,
				serviceName: 'srvhealth_restaurant_mgmt_select',
				defaultVal: item
			};
			uni.navigateTo({
				url: '/publicPages/newForm/newForm?serviceName=srvhealth_restaurant_mgmt_select&type=update&fieldsCond=' + encodeURIComponent(JSON.stringify(cond))
			});
		},
		/*删除**/
		del(item) {
			let self = this;
			uni.showModal({
				title: '提示',
				content: '是否确认删除?',
				success: res => {
					if (res.confirm) {
						self.delFoods(item);
					} else if (res.cancel) {
						console.log('用户点击取消');
					}
				}
			});
		},
		async delFoods(item) {
			let self = this;
			let url = this.getServiceUrl('health', 'srvhealth_restaurant_mgmt_delete', 'operate');
			let req = [
				{
					serviceName: 'srvhealth_restaurant_mgmt_delete',
					colNames: ['*'],
					condition: [
						{
							colName: 'id',
							ruleType: 'in',
							value: item.id
						}
					]
				}
			];
			let rea = await self.$http.post(url, req);
			if (rea.data.resultCode === 'SUCCESS') {
				// self.getFoodsList()
				self.getMyShopList();
			} else {
				uni.showToast({
					title: rea.data.resultMessage,
					icon: 'none'
				});
			}
		},
		async getuserinfo(e) {
			// #ifdef MP-WEIXIN
			const user = e.mp.detail;
			if (user && user.userInfo) {
				let rawData = {
					nickname: user.userInfo.nickName,
					sex: user.userInfo.gender,
					country: user.userInfo.country,
					province: user.userInfo.province,
					city: user.userInfo.city,
					headimgurl: user.userInfo.avatarUrl
				};
				this.setWxUserInfo(rawData);
				this.$store.commit('SET_WX_USERINFO', rawData);
				this.$store.commit('SET_AUTH_SETTING', { type: 'userInfo', value: true });
				this.$store.commit('SET_AUTH_USERINFO', true);
				const result = await wx.login();
				if (result.code) {
					this.wxLogin({
						code: result.code
					});
					this.initPage();
				}
			}
			// #endif
		},
		goSearch() {
			if (this.current_tit.type === 'shop') {
				this.getShopList('search', this.searchVal);
			} else if (this.current_tit.type === 'myShop') {
				this.getMyShopList('search', this.searchVal);
			}
			console.log('-=========>');
		},
		/* 点击顶部切换商铺和我得商铺列表**/
		tapTitItem(item, i) {
			this.current_tit = item;
			this.sortIndex = i;
			this.onRefresh();
			console.log('点击顶部切换===》', item);
		},
		/*跳转至商铺详情*/
		toShopDetail(item) {
			debugger;
			uni.navigateTo({
				url: '/otherPages/shop/shopHome?type=' + this.current_tit.type + '&restaurantNo=' + item.restaurant_no
			});
		},
		/* 获取商户列表**/
		async getShopList(type = null, search_val) {
			let self = this;
			let url = this.getServiceUrl('health', 'srvhealth_restaurant_mgmt_select', 'select');
			let req = {
				serviceName: 'srvhealth_restaurant_mgmt_select',
				colNames: ['*'],
				condition: [],
				order: [],
				page: self.pageInfo
			};
			if (type && type === 'search') {
				req.condition = [
					{
						colName: 'name',
						ruleType: 'like',
						value: search_val
					}
				];
			}
			let res = await this.$http.post(url, req);
			if (res.data.resultCode === '0011') {
				this.isLogin = false;
				const result = await wx.login();
				if (result.code) {
					this.code = result.code;
					await this.wxLogin({ code: result.code });
					this.isLogin = true;
					// await this.initPage();
				}
			} else {
				if (self.pageInfo.pageNo === 1) {
					self.storeList = [];
				}
				self.pageInfo.total = res.data.page.total;
				self.pageInfo.pageNo = res.data.page.pageNo;
				let page = self.pageInfo;
				if (page.rownumber * page.pageNo >= page.total) {
					// finish(boolean:是否显示finishText,默认显示)
					self.$refs.pullScroll.finish();
				} else {
					self.$refs.pullScroll.success();
				}
				if (res.data.state === 'SUCCESS') {
					console.log('商户列表-----', res.data.data);
					res.data.data.forEach(item => {
						if (item.image) {
							let urls = self.$api.downloadFile + item.image + '&bx_auth_ticket=' + uni.getStorageSync('bx_auth_ticket') + '&thumbnailType=fwsu_100';
							this.$set(item, 'imgurl', urls);
						}
					});
					this.storeList = [...this.storeList, ...res.data.data];
				}
			}
		},
		/* 获取当前登录人得商铺**/
		async getMyShopList(type = null, search_val) {
			let self = this;
			let url = this.getServiceUrl('health', 'srvhealth_restaurant_mgmt_select', 'select');
			let req = {
				serviceName: 'srvhealth_restaurant_mgmt_select',
				colNames: ['*'],
				condition: [
					{
						colName: 'create_user',
						ruleType: 'eq',
						value: uni.getStorageSync('login_user_info').user_no
					}
				],
				order: [],
				page: self.pageInfo
			};
			if (type && type === 'search') {
				let obj = {
					colName: 'name',
					ruleType: 'like',
					value: search_val
				};
				req.condition.push(obj);
			}
			let res = await this.$http.post(url, req);
			if (self.pageInfo.pageNo === 1) {
				self.myStoreList = [];
			}
			self.pageInfo.total = res.data.page.total;
			self.pageInfo.pageNo = res.data.page.pageNo;
			let page = self.pageInfo;
			if (page.rownumber * page.pageNo >= page.total) {
				// finish(boolean:是否显示finishText,默认显示)
				self.$refs.pullScroll.finish();
			} else {
				self.$refs.pullScroll.success();
			}

			if (res.data.state === 'SUCCESS') {
				console.log('我的商户列表-----', res.data.data);
				res.data.data.forEach(item => {
					if (item.image) {
						let urls = self.$api.downloadFile + item.image + '&bx_auth_ticket=' + uni.getStorageSync('bx_auth_ticket') + '&thumbnailType=fwsu_100';
						this.$set(item, 'imgurl', urls);
					}
				});
				this.myStoreList = [...this.myStoreList, ...res.data.data];
				if (this.myStoreList.length == 0) {
					this.current_tit = { tit: '商户', type: 'shop' };
					this.getShopList();
				}
			}
		},
		/*新增商铺**/
		addShop() {
			uni.navigateTo({
				url: '/publicPages/newForm/newForm?serviceName=srvhealth_restaurant_mgmt_add&type=add'
			});
		}
	}
};
</script>

<style lang="scss">
.store-wrap {
	height: 100%;
}
page {
	background-color: #f8f8f8;
}
.b-b {
	position: relative;
}
.b-b:after,
.b-t:after {
	position: absolute;
	z-index: 3;
	left: 0;
	right: 0;
	height: 0;
	content: '';
	transform: scaleY(0.5);
	border-bottom: 1px solid #e4e7ed;
}

.b-b:after {
	bottom: 0;
}
.b-t:after {
	top: 0;
}
.container {
	margin: 0 15px;
}
.margin-top {
	margin-top: 12px;
}
.head-box {
	padding: 14px 0;
	/* #ifdef MP */
	padding-top: 2px;
	/* #endif */
	background: linear-gradient(100deg, #ffeb3b, #ffc107);
	position: relative;
	z-index: 3;
	.navbar {
		position: sticky;
		top: 0;
		// height: var(--status-bar-height);
	}

	.container {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		height: 32px;
		align-items: center;
		.left {
			.logo {
				width: 32px;
				height: 32px;
				image {
					width: 32px;
					height: 32px;
					border-radius: 50%;
				}
			}
		}
		.right {
			display: flex;
			flex-direction: column;
			color: #333333;
			justify-content: space-around;
			.address-box {
				font-size: 14px;
				margin-right: 16px;
				display: flex;
				flex-direction: row;
				align-items: center;
				text {
					margin-left: 2px;
					margin-right: 4px;
				}
				.icon-right {
					position: relative;
					top: 2px;
				}
			}

			.notice {
				font-weight: bold;
				font-size: 18px;
			}
		}
	}
}
.store-r-b {
	display: flex;
	align-items: center;
	.store-r-b-t {
		color: #f98c00;
		font-weight: 700;
		display: flex;
		text {
			margin-right: 10upx;
		}
	}
	.store-r-b-b {
		color: #848484;
		// margin-top: 10upx;
	}
}
.store-r-b-my {
	margin-top: 30upx;
}

.hot-box {
	display: flex;
	flex-direction: row;
	flex-wrap: wrap;
	justify-content: flex-start;
	.item {
		margin-right: 12px;
		background-color: #eeeeee;
		color: #666666;
		font-size: 12px;
		border-radius: 20px;
		padding: 2px 6px;
		margin-bottom: 8px;
	}
	.item:last-child {
		margin-right: 0;
	}
}
.bannerimg-box {
	border-bottom-left-radius: 10upx;
	border-bottom-right-radius: 10upx;
	padding: 24rpx;
	swiper {
		height: 233rpx;
		width: 699rpx;
	}
	.swiper-item {
		display: flex;
		justify-content: center;
		align-content: center;
		overflow: hidden;

		width: 100%;
		height: 100%;
		image {
			border-radius: 8px;
			width: 100%;
		}
	}
}

.menu-box {
	display: flex;
	flex-direction: row;
	justify-content: space-between;
	margin-top: 12px;
	border-radius: 8px;
	background: #ffffff;
	padding: 12px 8px;
	.item-box {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		image {
			width: 40px;
			height: 40px;
		}
		.tit {
			display: flex;
			justify-content: center;
			align-items: center;
			font-size: 12px;
			margin-top: 6px;
			color: #333333;
			padding: 2px 0;
			white-space: nowrap;
			overflow: hidden;
			text-overflow: ellipsis;
		}
		.active {
			color: #ffffff;
			background-color: #999999;
			border-radius: 60px;
		}
	}
}
.sort-box {
	display: flex;
	flex-direction: row;
	margin-top: 12px;
	.item-box {
		margin-right: 16px;
		.tit {
			display: flex;
			justify-content: center;
			align-items: center;
			font-size: 12px;
			color: #333333;
			white-space: nowrap;
			overflow: hidden;
			text-overflow: ellipsis;
		}
		.active {
			//color: #111111;
			font-weight: bold;
		}
	}
}
.store-box {
	margin-top: 20px;

	.item-box {
		display: flex;
		flex-direction: column;
		// margin-bottom: 14px;
		border-bottom: 1px solid #ccc;
		padding-bottom: 18px;
		padding-top: 18px;
		background-color: #ffffff;
		.top-box {
			display: flex;
			flex-direction: row;
			.left {
				margin-right: 10px;
				image {
					width: 150upx;
					height: 150upx;
					border-radius: 6px;
				}
			}
			.right {
				flex: 1;
				display: flex;
				flex-direction: column;
				justify-content: space-between;
				.tit {
					font-size: 16px;
					font-weight: bold;
					color: #333333;
					white-space: nowrap;
					overflow: hidden;
					text-overflow: ellipsis;
				}
				.justify-content {
					display: flex;
					flex-direction: row;
					justify-content: space-between;
				}
				.row {
					font-size: 12px;
					color: #999999;
					margin-top: 4px;
					.row-left {
						display: flex;
						flex-direction: row;
						.t1,
						i {
							color: #ffca3e;
						}
					}
					.t2 {
						margin-left: 12px;
					}
					.row-right {
						display: flex;
						flex-direction: row;
						i {
							margin-right: 4px;
						}
					}
				}
			}
		}
		.bottom-box {
			&-container {
				margin: 12px 0 0;
				padding-left: 82px;

				white-space: nowrap;
				.goods-box {
					width: 80px;
					margin-left: 12px;
					display: inline-block;
					.img-box {
						position: relative;
						width: 80px;
						height: 60px;
						background-color: #f1f1f1;
						border-radius: 4px;
						image {
							width: 100%;
							height: 60px;
						}
						.tag {
							position: absolute;
							bottom: 0;
							left: 0;
							background-color: #ff5722;
							color: #ffffff;
							padding: 2px 4px;
							font-size: 12px;
							border-radius: 4px;
						}
					}
					.tit {
						font-size: 14px;
						margin-top: 4px;
						white-space: nowrap;
						overflow: hidden;
						text-overflow: ellipsis;
					}
					.price-box {
						margin-top: 4px;
						height: 80upx;
						.txt1 {
							font-size: 10px;
							color: #ff5722;
						}
						.txt2 {
							font-size: 16px;
							color: #ff5722;
						}
						.txt3 {
							margin-left: 6px;
							font-size: 10px;
							color: #bbbbbb;
							text-decoration: line-through;
						}
					}
				}
				.goods-box:last-child {
					margin-right: 12px;
				}
			}
		}
		.del-shop {
			display: flex;
			justify-content: flex-end;
			margin-right: 20upx;
			text {
				color: #0081ff;
				margin-right: 10px;
				border: 1px solid #0081ff;
				border-radius: 20px;
				padding: 2px 10px;
			}
		}
	}
	.item-box:last-child {
		border-bottom: 0;
	}
}
.foot {
	position: relative;
	display: flex;
	justify-content: center;
	margin-top: 36px;
	margin-bottom: 50px;
	text {
		font-size: 14px;
		position: relative;
		z-index: 2;
		height: 20px;
		line-height: 20px;
		background-color: #f8f8f8;
		color: #cccccc;
		padding: 0 12px;
	}
}
.foot::before {
	content: '';
	z-index: 1;
	display: block;
	position: absolute;
	top: 50%;
	height: 0;
	width: 100%;
	transform: scaleY(0.5);
	border-bottom: 1px solid #e4e7ed;
}
.footzw {
	/* #ifdef H5*/
	height: 50px;
	/* #endif */
}
.add-button {
	position: fixed;
	bottom: 20upx;
	left: calc(50% - 50upx);
	width: 100upx;
	height: 100upx;
	border-radius: 50%;
	background-color: rgb(246, 210, 0);
	z-index: 100;
	display: flex;
	justify-content: center;
	align-items: center;
	font-size: 24px;
	color: white;
}
</style>
