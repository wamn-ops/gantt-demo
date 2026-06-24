<template>
    <div class="chart-container">
        <!-- Chart Filter -->
        <div class="chart-filter">
            <div class="filter-item">
                <label>成员：</label>
                <Select v-model:value="searchParams.members" mode="multiple" placeholder="请选择成员"
                    style="min-width: 220px" :max-tag-count="2" allow-clear :options="memberOptions" />
            </div>
            <div class="filter-item">
                <label>日期范围：</label>
                <RangePicker v-model:value="dateRange" value-format="YYYY-MM-DD" style="min-width: 240px" />
            </div>
            <Button type="primary" @click="handleSearch">
                <template #icon>
                    <SearchOutlined />
                </template>
                查询
            </Button>
            <Button @click="handleReset">重置</Button>
        </div>
        <!-- Chart Body -->
        <div class="chart-wrapper">
            <div class="chart-inner" :style="{ minWidth: totalWidth + 'px' }">
                <!-- 表头 -->
                <div class="chart-head">
                    <!--  月份 -->
                    <div class="chart-row">
                        <div class="row-cell"></div>
                        <div class="head-cell" v-for="(item, mi) in monthData" v-show="item.days > 0" :key="mi"
                            :style="{ width: item.days * cellWidth + 'px' }"
                            :class="{ 'col-hovered': hoveredColIndex !== null && hoveredColIndex >= getMonthRange(mi).start && hoveredColIndex <= getMonthRange(mi).end }">{{ item.label }}</div>
                    </div>
                    <!--  日期 -->
                    <div class="chart-row">
                        <div class="row-cell"></div>
                        <div class="head-cell" v-for="(item, di) in dateData" :key="di"
                            :style="{ width: cellWidth + 'px' }"
                            :class="{ 'col-hovered': hoveredColIndex === di }">
                            {{ item.day }}
                        </div>
                    </div>
                </div>
                <!-- 表体 数据行 -->
                <div class="chart-body">
                    <div class="data-row" v-for="(row, i) in visibleRows" :key="i"
                        @mouseenter="hoveredRowIndex = i" @mouseleave="hoveredRowIndex = null"
                        :class="{ 'row-hovered': hoveredRowIndex === i }">
                        <div  class="data-username">
                           <div v-if="row.isFirstTask">
                                <CaretRightOutlined v-if="row.taskCount > 3" @click="handleExpand(row)" :style="{ transform: collapsedUsers.has(row.id) ? 'rotate(0deg)' : 'rotate(90deg)' }" />
                                {{ row.userName }}
                            </div>
                        </div>
                        <!--格子  -->
                        <div class="data-grid">
                            <div class="grid-cell" v-for="(day, dayIndex) in dateData" :key="dayIndex"
                                :style="{ width: cellWidth + 'px' }"
                                @mouseenter="hoveredColIndex = dayIndex" @mouseleave="hoveredColIndex = null"
                                :class="{ 'col-hovered': hoveredColIndex === dayIndex }">
                            </div>
                            <div class="task-info" :style="figureStyle(row.task)">
                                <div class="progress-bar" :style="{ width: row.task.progress + '%' }"></div>
                                <div class="task-txt" :title="row.task.progress + '%' + row.task.title">
                                    {{ row.task.progress }}% {{ row.task.title }}
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<script setup>
import { ref, onMounted, computed, watch } from 'vue'
import { Button, Select, DatePicker } from 'ant-design-vue'
import { SearchOutlined,CaretRightOutlined } from '@ant-design/icons-vue'
const { RangePicker } = DatePicker;

const searchParams = ref({
    members: [],
    startDate: '2026-09-01',
    endDate: '2026-10-01',
})
const cellWidth = 40; // 每个格子的宽度
const dateRange = ref([searchParams.value.startDate, searchParams.value.endDate]);
const collapsedUsers = ref(new Set()); // 用于存储折叠的用户
const hoveredRowIndex = ref(null);
const hoveredColIndex = ref(null);

const handleReset = () => {
    searchParams.value.members = [];
    searchParams.value.startDate = '2026-09-01';
    searchParams.value.endDate = '2026-10-01';
    dateRange.value = ['2026-09-01', '2026-10-01'];
    collapsedUsers.value = new Set();
};

const memberOptions = computed(() => {
    return (filterData.value.users || []).map(user => ({
        label: user.name,
        value: user.id
    }));
});

watch(dateRange, (val) => {
  if (val && val.length === 2) {
    searchParams.value.startDate = val[0];
    searchParams.value.endDate = val[1];
  }
});

// 请求数据源
const filterData = ref([]);
const fetchGanttData = async () => {
    const data = await import('../mock/data.json');
    return data.default || data;
};

// 获取所选日期范围
const dateData = computed(()=>{
  const startDate = new Date(searchParams.value.startDate);
  const endDate = new Date(searchParams.value.endDate);
  const dates = [];
  const currentDate = new Date(startDate);
  while (currentDate <= endDate) {
        dates.push({
            day: currentDate.getDate(),
            month: currentDate.getMonth() + 1,
            year: currentDate.getFullYear(),
            dateStr: currentDate.toISOString().split('T')[0]
        })
        currentDate.setDate(currentDate.getDate() + 1)
    }
    return dates
})

// 根据日期范围获取月份数据
const monthData = computed(()=>{
    const monthMp= dateData.value.reduce((acc,cur)=>{
        const momthKey = `${cur.year}-${cur.month}`;
        if(!acc[momthKey]){
            acc[momthKey] = {
                label: `${cur.year}-${cur.month}`,
                key: momthKey,
                days: 0
            }
        } 
        acc[momthKey].days += 1;
        return acc
    })
    return Object.values(monthMp)
})

// 图表最小宽度，确保日期列过多时出现横向滚动条
const totalWidth = computed(() => 80 + dateData.value.length * cellWidth);

// 获取指定月份对应的日期列范围，用于月份单元格的列高亮
const getMonthRange = (mi) => {
    let start = 0;
    for (let i = 0; i < mi; i++) start += monthData.value[i].days;
    return { start, end: start + monthData.value[mi].days - 1 };
};

// 人员筛选结果
const userData = computed(()=>{
    const userData = filterData.value.users||[]
    if(!searchParams.value.members?.length){
        return userData
    }
    return userData.filter(user => searchParams.value.members.includes(user.id))

})

// 将任务数据拍平展示
const flattenedRows = computed(()=>{
    const rows = [];
    userData.value.forEach(user => {
        user.tasks.forEach((task, index) => {
            rows.push({
                userName: user.name,
                id:user.id,
                task: task,
                isFirstTask: index === 0,
                taskCount: user.tasks.length
            });
        });
    });
    return rows;
})


// 根据筛选条件获取可见的行数据
const visibleRows = computed(()=>{
   return flattenedRows.value.filter(item =>{
    if(collapsedUsers.value.has(item.id)){
        // 只返回第一条任务
        return item.isFirstTask
    }
    return true
   })
})

// 计算任务条的样式
const figureStyle = (task) => {
    const beginIndex = dateData.value.findIndex(d => d.dateStr === task.startDate);
    const endIndex = dateData.value.findIndex(d => d.dateStr === task.endDate);
    if (beginIndex === -1 || endIndex === -1) {
        return { left: '0px', width: '0px' };
    }
    const left = beginIndex * cellWidth;
    const width = (endIndex - beginIndex + 1) * cellWidth;
    return { left: left + 'px', width: width + 'px' };
}

//展开和收起
const handleExpand=row =>{
    const id = row.id;
    const collapsedSet = new Set(collapsedUsers.value);
    if(collapsedSet.has(id)){
        collapsedSet.delete(id);
    }else{
        collapsedSet.add(id);
    }
    collapsedUsers.value = collapsedSet;
}

onMounted(async () => {
    filterData.value = await fetchGanttData();
});
</script>
<style>
.chart-container{
    height: 100%;
    display: flex;
    flex-direction: column;
    overflow: hidden;
}
/* ========== 筛选栏 ========== */
.chart-filter {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 10px 16px;
  background: #fafafa;
  border-bottom: 1px solid #e0e0e0;
  flex-wrap: wrap;
}
.filter-item {
  display: flex;
  align-items: center;
  gap: 8px;
}
.filter-item label {
  font-size: 13px;
  font-weight: 5    00;
  white-space: nowrap;
  color: #333;
}

/* ========= 表格 ========= */
.chart-wrapper{
    overflow: auto;
    flex: 1;
}
.chart-inner{
    display: flex;
    flex-direction: column;
    height: 100%;
}
.chart-head{
    position: sticky; /* sticky 表示 */
    top: 0;
    background: #f5f5f5;
    z-index: 9;
}
.head-cell{
  flex-shrink: 0;
  text-align: center;
  font-size: 12px;
  border-right: 1px solid #e0e0e0;
  border-bottom: 1px solid #e0e0e0;
  padding: 4px 0;
  box-sizing: border-box;
  transition: background-color 0.15s;
}
.chart-row {
  display: flex;
}
.row-cell {
  flex-shrink: 0;
  width: 80px;
  /* padding-left: 4px; */
  border-right: 1px solid #e0e0e0;
  border-bottom: 1px solid #e0e0e0;
  background: #fafafa;
}
.chart-head .row-cell {
  position: sticky;
  left: 0;
  z-index: 10;
}
/* 表体 */

.chart-body{
    /* border:1px solid red; */
}
.data-row{
    display:flex;
}
.data-username{
  position: sticky;
  left: 0;
  z-index: 9;
  flex-shrink: 0;
  font-size: 13px;
  font-weight: 500;
  border-right: 1px solid #e0e0e0;
  border-bottom: 1px solid #f5f5f5;
  min-width: 80px;
  height: 36px;
  display: flex;
  align-items: center;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  background: #fff;
  box-sizing: border-box;
  transition: background-color 0.15s;
}
/* 日期网格 */
.data-grid{
    display:flex;
    position: relative;
    /* margin-left: 80px; */
    height: 36px;
}
.grid-cell{
    border-right: 1px solid #eee;
    border-bottom: 1px solid #f5f5f5;
    height: 36px;
    flex-shrink: 0;
    transition: background-color 0.15s;
    box-sizing: border-box;
}
.task-info{
    position: absolute;
    top: 5px;
    height: 26px;
    background: #ffc107;
    border-radius: 4px;
    overflow: hidden;
    cursor: pointer;
    z-index: 2;
}
.progress-bar{
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    background: #4caf50;
    border-radius: 4px;
    transition: width 0.3s ease;
}
.task-txt{
    position: relative;
    left: 4px;
    font-size: 12px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
/* ========= 行悬停高亮 ========= */
.data-row.row-hovered .data-username,
.data-row.row-hovered .grid-cell {
  background-color: rgba(0, 0, 0, 0.04);
}
/* ========= 列悬停高亮 ========= */
.head-cell.col-hovered,
.grid-cell.col-hovered {
  background-color: rgba(0, 0, 0, 0.04);
}
</style>