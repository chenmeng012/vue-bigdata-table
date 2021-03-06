<style lang="less">
	@import './vue-bigdata-table.less';
</style>

<template>
	<div class="vue-bigdata-table-outer" ref="outer" @DOMMouseScroll="handleScroll" @scroll="handleScroll">
		<div :class="wrapperClasses" :style="tableWidthStyles">
			<div class="vue-bigdata-table-wrapper" ref="outWrapper">
				<div :class="['vue-bigdata-table-header-wrapper', fixed ? 'header-wrapper-fixed' : '']" :style="headerStyle">
					<slot name="top" :colWidthArr="widthArr"></slot>
					<table v-if="fixedCol >= 0" :class="['vue-bigdata-table-fixed-header', showFixedBoxShadow ? 'box-shadow' : '']" cellspacing="0" cellpadding="0" border="0">
						<colgroup>
							<col v-if="i <= fixedCol" :width="width" v-for="(width, i) in widthArr" :key="'header-key-fixed-' + i" />
						</colgroup>
						<tr
							:style="{cursor: cursorOnHeader}"
							:data-update="updateID"
							@mousemove.capture.prevent="handleMousemove"
							@mousedown="handleMousedown"
							@mouseup="canNotMove"
							@mouseleave="canNotMove">
							<th v-if="i <= fixedCol" v-for="(col, i) in columnsHandled" :data-index="i" :key="`table-title-${i}`" style="border-right: 1px solid #e9eaec;">
								<span v-if="!col.render">{{ col.title }}<sort-button v-if="showSortBtn(i)" :col-index="i" @on-sort="handleSort" @on-cancel-sort="handleCancelSort" :current-sort-col-index="sortedByColIndex" :current-sort-type="sortedType"></sort-button></span>
								<render-dom v-else :render="col.render" :back-value="getComputedTableDataIndex(i)"></render-dom>
							</th>
						</tr>
					</table>
					<table ref="headerTable" style="position: absolute;left: 0;top: 0;" cellspacing="0" cellpadding="0" border="0" width="100%">
						<colgroup>
							<col :width="width" v-for="(width, i) in widthArr" :key="'header-key-' + i" />
						</colgroup>
						<tr
							:style="{cursor: cursorOnHeader}"
							:data-update="updateID"
							@mousemove.capture.prevent="handleMousemove"
							@mousedown="handleMousedown"
							@mouseup="canNotMove"
							@mouseleave="canNotMove">
							<th v-for="(col, i) in columnsHandled" :data-index="i" :key="`table-title-${i}`">
								<span v-if="!col.render && (i > fixedCol)">{{ col.title }}<sort-button v-if="showSortBtn(i)" :col-index="i" @on-sort="handleSort" @on-cancel-sort="handleCancelSort" :current-sort-col-index="sortedByColIndex" :current-sort-type="sortedType"></sort-button></span>
								<render-dom v-else-if="(i > fixedCol)" :render="col.render" :back-value="getComputedTableDataIndex(i)"></render-dom>
							</th>
						</tr>
					</table>
				</div>
				<div class="vue-bigdata-table-content" @mousedown="handleMousedownOnTable">
					<div :style="{height: `${topPlaceholderHeight}px`}"></div>
					<render-dom :render="renderTable"></render-dom>
					<div :style="{height: `${bottomPlaceholderHeight}px`}"></div>
				</div>
			</div>
		</div>
	</div>
</template>
<script>
import renderDom from './components/renderDom';
import sortButton from './components/sort-button.vue';
import editRender from './components/input-render';
import mixins from './mixins';
export default {
	name: 'bigdataTable',
	components: {
		renderDom,
		sortButton
	},
	mixins: [ ...mixins ],
	props: {
		/**
		 * @description 是否显示序列号列
		 */
		showIndex: {
			type: Boolean,
			default: false
		},
		rowNum: Number,
		colNum: Number,
		/**
		 * @description 表格数据二维数组
		 */
		value: {
			type: Array
		},
		/**
		 * @description 表格行高
		 */
		rowHeight: {
			type: Number,
			default: 48
		},
		/**
		 * @description 是否固定表头
		 */
		fixed: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 设为true后表格列宽总是平分容器宽度减去indexWidth后的宽度
		 */
		fixedWrapperWidth: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 是否取消鼠标悬浮高亮效果
		 */
		disabledHover: {
			type: Boolean,
			default: true
		},
		/**
		 * @description 点击一行是否高亮
		 */
		highlightRow:{
			type: Boolean,
			default: false
		},
		/**
		 * @description 表头数组，元素为单个表头的对象，【{ title: 'xxx', render: (h) => {} }】
		 */
		columns: {
			type: Array
		},
		/**
		 * @description 列宽，如果单独列中指定了宽度则按单独列，如果所有宽度加起来比容器宽度小，则平分宽度，否则用colWidth
		 */
		colWidth: {
			type: Number,
			default: 100
		},
		/**
		 * @description 表头高度
		 */
		headerHeight: {
			type: Number,
			default: 52
		},
		/**
		 * @description 表头tr行的样式
		 */
		headerTrStyle: {
			type: Object,
			default: () => {
				return {};
			}
		},
		/**
		 * @description 序列号列宽，如果没有设置，则会根据数据行数自动计算适合的宽度
		 */
		indexWidth: Number,
		/**
		 * @description 序列号渲染render
		 */
		indexRender: {
			type: Function,
			default: (h, params) => {
				return h('span', params.index + 1);
			}
		},
		/**
		 * @description indexRender的第三个参数
		 */
		indexRenderParams: {
			type: Object,
			default: () => {
				return {};
			}
		},
		/**
		 * @description 是否显示斑马线
		 */
		stripe: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 当前鼠标在表头单元格左侧atLeftCellPosi像素处
		 */
		atLeftCellPosi: {
			type: Number,
			default: 80
		},
		/**
		 * @description 当前鼠标在表头单元格右侧atRightCellPosi像素处
		 */
		atRightCellPosi: {
			type: Number,
			default: 80
		},
		/**
		 * @description 固定的列的范围，[0, fixedCol]，设为2即固定0，1，2列，这三列横向不滚动
		 */
		fixedCol: {
			type: Number,
			default: -1
		},
		/**
		 * @description 根据表格容器高度计算内置单个表格（1/3）渲染的行数基础上额外渲染的行数，行数越多表格接替渲染效果越好，但越耗性能
		 */
		appendNum: {
			type: Number,
			default: 15
		},
		/**
		 * @description 当前是否可编辑
		 */
		canEdit: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 触发编辑单元格的方式，enum:['dblclick' => 双击单元格]
		 */
		startEditType: {
			type: String,
			default: 'dblclick'
		},
		/**
		 * @description 编辑单元格所渲染元素的render函数，如果不传则使用内置元素
		 */
		editCellRender: {
			type: Function,
			default: editRender
		},
		/**
		 * @description 保存修改的单元格内容之前的钩子，如果该函数返回false，则阻止保存
		 */
		beforeSave: {
			type: Function,
			default: () => {
				return true;
			}
		},
		/**
		 * @description 是否可选择单元格
		 */
		selectable: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 是否可粘贴，如果设为true，则selectable效果为true
		 */
		paste: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 是否可排序
		 */
		sortable: {
			type: Boolean,
			default: false
		},
		/**
		 * @description 开启排序的列序号数组或序号
		 */
		sortIndex: [Array, Number],
		/**
		 * @description 默认按指定列指定排序方式排序
		 */
		defaultSort: Object
	},
	/**
	 * Events
	 * @on-click-tr 	des: 点击行时触发
	 * 					arg: index: 当前行号, initRowIndex: 初始行号（排序筛选之前）
	 * @on-click-td 	des: 点击单元格触发
	 * 					arg: { row: 行号, col: 列号 }
	 * @on-success-save des: 编辑成功时触发
	 * 					arg: { row: 行号, col: 列号, value: 修改后的值, initRowIndex: 初始行号 }
	 * @on-fail-save    des: 编辑失败时触发
	 * 					arg: 同on-success-save
	 * @on-paste		des: 粘贴数据成功时触发
	 * 					des: data: 粘贴的数据的二维数组
	 */
	data () {
		return {
			prefix: 'vue-bigdata-table'
		};
	},
	methods: {
		// 涉及到表格容器尺寸变化或数据变化的情况调用此方法重新计算相关值
		resize (changeInitIndex) {
			// this.insideTableData = [...this.value]
			this.$nextTick(() => {
				if (changeInitIndex) this.insideTableData = this.setInitIndex(this.value);
				else this.insideTableData = [...this.value];
				this.initSort();
				// this._initMountedHandle();
			});
			// this._tableResize();
		},
		// 获取表格横向滚动的距离
		getScrollLeft () {
			return this.$refs.outer.scrollLeft;
		},
		// 调用此方法跳转到某条数据
		scrollToRow (row) {
			this._scrollToIndexRow(row);
		},
		// canEdit为true时调用此方法使第row+1行第col+1列变为编辑状态，这里的行列指的是表格显示的行和除序列号列的列
		editCell (row, col, scrollToView) {
			this._editCell(row, col, scrollToView);
		},
		// canEdit为true时调用此方法使指定单元格被选中
		selectCell (row, col) {
			this._selectCell(row, col);
		},
		// 手动设置高亮行
		setHighlightRow (row) {
			this._setHighlightRow(row);
		},
		/**
		 * @argument {Number} col 要按哪一列筛选的列号
		 * @argument {Array} queryArr 筛选关键字数组
		 * @description 按照某一列的指定关键词进行筛选
		 */
		filter (col, queryArr) {
			this._filter(col, queryArr);
		},
		/**
		 * @description 取消筛选
		 */
		cancelFilter () {
			this._cancelFilter();
		},
		undo () {
			this._undo();
		},
		/**
		 * @description 清除高亮项目
		 */
		clearCurrentRow () {
			this._clearCurrentRow();
		},
		/**
		 * @description 获取指定行的初始行号
		 */
		getInitRowIndexByIndex (row) {
			return this._getInitRowIndexByIndex(row);
		},
		/**
		 * @description 获取指定初始行号的当前行号
		 */
		getIndexByInitRowIndex (initRow) {
			return this._getIndexByInitRowIndex(initRow);
		}
	},
	watch: {
		value () {
			this.$nextTick(() => {
				this.insideTableData = this.setInitIndex(this.value);
				this.initSort();
				this._initMountedHandle();
			});
		},
		insideTableData () {
			this._tableResize();
		},
		defaultSort () {
			this.insideTableData = this.setInitIndex(this.value);
			this._initMountedHandle();
			this.resize();
		}
	},
	mounted () {
		this.$nextTick(() => {
			this.insideTableData = this.setInitIndex(this.value);
			this._initMountedHandle();
			this.resize();
		});
	}
};
</script>
