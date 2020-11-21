<template>
	<view class="bx-checkbox" :style="[checkboxStyle]">
		<view class="bx-checkbox-icon" @tap="toggle" :class="{ checked: value }">
			<text v-if="serialChar">{{ serialChar }}</text>
			<text class="cuIcon-check" v-if="value && !serialChar"></text>
		</view>
		<view class="bx-checkbox__label" @tap="onClickLabel"><slot /></view>
	</view>
</template>

<script>
/**
 * checkbox 复选框
 * @description 该组件需要搭配checkboxGroup组件使用，以便用户进行操作时，获得当前复选框组的选中情况。
 * @tutorial https://www.uviewui.com/components/checkbox.html
 * @property {String Number} icon-size 图标大小，单位rpx（默认20）
 * @property {String Number} label-size label字体大小，单位rpx（默认28）
 * @property {String Number} name checkbox组件的标示符
 * @property {String} shape 形状，见官网说明（默认circle）
 * @property {Boolean} disabled 是否禁用
 * @property {Boolean} label-disabled 是否禁止点击文本操作checkbox
 * @property {String} active-color 选中时的颜色，如设置CheckboxGroup的active-color将失效
 * @event {Function} change 某个checkbox状态发生变化时触发，回调为一个对象
 * @example <bx-checkbox v-model="checked" :disabled="false">天涯</bx-checkbox>
 */
export default {
	name: 'bx-checkbox',
	props: {
		// checkbox的名称
		name: {
			type: [String, Number],
			default: ''
		},
		// 形状，square为方形，circle为原型
		shape: {
			type: String,
			default: ''
		},
		// 是否为选中状态
		value: {
			type: Boolean,
			default: false
		},
		// 是否禁用
		disabled: {
			type: [String, Boolean],
			default: ''
		},
		// 是否禁止点击提示语选中复选框
		labelDisabled: {
			type: [String, Boolean],
			default: ''
		},
		// 选中状态下的颜色，如设置此值，将会覆盖checkboxGroup的activeColor值
		activeColor: {
			type: String,
			default: ''
		},
		// 图标的大小，单位rpx
		iconSize: {
			type: [String, Number],
			default: ''
		},
		// label的字体大小，rpx单位
		labelSize: {
			type: [String, Number],
			default: ''
		},
		// 组件的整体大小
		size: {
			type: [String, Number],
			default: ''
		},
		// 序号
		serialChar: {
			type: [String, Number],
			default: ''
		}
	},
	data() {
		return {
			parentDisabled: false,
			newParams: {}
		};
	},
	created() {
		this.parent = this.theParent.call(this, 'bx-checkbox-group');
		// 如果存在bx-checkbox-group，将本组件的this塞进父组件的children中
		this.parent && this.parent.children.push(this);
	},
	computed: {
		// 是否禁用，如果父组件bx-checkbox-group禁用的话，将会忽略子组件的配置
		isDisabled() {
			return this.disabled !== '' ? this.disabled : this.parent ? this.parent.disabled : false;
		},
		// 是否禁用label点击
		isLabelDisabled() {
			return this.labelDisabled !== '' ? this.labelDisabled : this.parent ? this.parent.labelDisabled : false;
		},
		// 组件尺寸，对应size的值，默认值为34rpx
		checkboxSize() {
			return this.size ? this.size : this.parent ? this.parent.size : 34;
		},
		// 组件的勾选图标的尺寸，默认20
		checkboxIconSize() {
			return this.iconSize ? this.iconSize : this.parent ? this.parent.iconSize : 20;
		},
		// 组件选中激活时的颜色
		elActiveColor() {
			return this.activeColor ? this.activeColor : this.parent ? this.parent.activeColor : 'primary';
		},
		// 组件的形状
		elShape() {
			return this.shape ? this.shape : this.parent ? this.parent.shape : 'square';
		},
		checkboxStyle() {
			let style = {};
			if (this.parent && this.parent.width) {
				style.width = this.parent.width;
				// #ifdef MP
				// 各家小程序因为它们特殊的编译结构，使用float布局
				style.float = 'left';
				// #endif
				// #ifndef MP
				// H5和APP使用flex布局
				style.flex = `0 0 ${this.parent.width}`;
				// #endif
			}
			if (this.parent && this.parent.wrap) {
				style.width = '100%';
				// #ifndef MP
				// H5和APP使用flex布局，将宽度设置100%，即可自动换行
				style.flex = '0 0 100%';
				// #endif
			}
			return style;
		}
	},
	methods: {
		theParent(name = undefined) {
			let parent = this.$parent;
			// 通过while历遍，这里主要是为了H5需要多层解析的问题
			while (parent) {
				// 父组件
				if (parent.$options && parent.$options.name !== name) {
					// 如果组件的name不相等，继续上一级寻找
					parent = parent.$parent;
				} else {
					return parent;
				}
			}
			return false;
		},
		onClickLabel() {
			if (!this.isLabelDisabled && !this.isDisabled) {
				this.setValue();
			}
		},
		toggle() {
			if (!this.isDisabled) {
				this.setValue();
			}
		},
		emitEvent() {
			this.$emit('change', {
				value: !this.value,
				name: this.name
			});
			// 执行父组件bx-checkbox-group的事件方法
			// 等待下一个周期再执行，因为this.$emit('input')作用于父组件，再反馈到子组件内部，需要时间
			setTimeout(() => {
				if (this.parent && this.parent.emitEvent) this.parent.emitEvent();
			}, 80);
		},
		// 设置input的值，这里通过input事件，设置通过v-model绑定的组件的值
		setValue() {
			// 判断是否超过了可选的最大数量
			let checkedNum = 0;
			if (this.parent && this.parent.children) {
				// 只要父组件的某一个子元素的value为true，就加1(已有的选中数量)
				this.parent.children.map(val => {
					if (val.value) checkedNum++;
				});
			}
			// 如果原来为选中状态，那么可以取消
			if (this.value == true) {
				this.emitEvent();
				this.$emit('input', !this.value);
			} else {
				// 如果超出最多可选项，提示
				if (this.parent && checkedNum >= this.parent.max) {
					uni.showToast({
						title: `最多可选${this.parent.max}项`,
						icon: 'none'
					});
					// return this.$u.toast(`最多可选${this.parent.max}项`);
				}
				// 如果原来为未选中状态，需要选中的数量少于父组件中设置的max值，才可以选中
				this.emitEvent();
				this.$emit('input', !this.value);
			}
		}
	}
};
</script>

<style lang="scss" scoped>
.bx-checkbox {
	display: flex;
	margin: 10rpx;
	min-width: calc(50% - 20rpx);
	align-items: center;
	&:nth-child(2n) {
		margin-right: 0;
	}
	.bx-checkbox-icon {
		border: 1rpx solid #888;
		width: 40rpx;
		height: 40rpx;
		justify-content: center;
		align-items: center;
		display: flex;
		border-radius: 10rpx;
		margin-right: 10rpx;
		font-size: 24rpx;
		&.checked {
			background-color: #0081ff;
			color: #fff;
			border-color: #fff;
			// color: #0081FF;
			border: none;
		}
	}
}
</style>