<template>
    <div style="width: 100vw;">
        <el-table :data="tableData" border ref="taskTableRef" class="tree-table-drag" :row-class-name="tableRowClassName"
            row-key="id" height="100vh">
            <el-table-column prop="date" label="Date" min-width="200px" />
            <el-table-column prop="name" label="Name" min-width="200px" />
            <el-table-column prop="address" label="Address" min-width="200px" />
            <el-table-column min-width="200px">
                <template #default="{ row }">
                    <el-button :icon="Rank" class="drag-mark" type="primary" size="small"
                        @mousedown="dragID = row.id"></el-button>
                </template>
            </el-table-column>
        </el-table>
    </div>
</template>

<script lang="ts" setup>
import { nextTick, onMounted, ref } from 'vue'
import { Rank } from '@element-plus/icons-vue'
import { cloneDeep } from 'lodash-es'
import Sortable, { MoveEvent, SortableEvent } from 'sortablejs'
import { useThrottleFn } from '@vueuse/core'
import { ElTable } from 'element-plus'


interface User {
    id: number
    date: string
    name: string
    address: string
    hasChildren?: boolean
    parentID: number,
    children?: User[]
}
const taskTableRef = ref<InstanceType<typeof ElTable>>() // Table Dom
let tableData = ref<User[]>([
    {
        id: 1,
        parentID: 0,
        date: '2016-05-02',
        name: 'wangxiaohu',
        address: 'No. 189, Grove St, Los Angeles',
    },
    {
        id: 2,
        parentID: 0,
        date: '2016-05-04',
        name: 'wangxiaohu',
        address: 'No. 189, Grove St, Los Angeles',
    },
    {
        id: 3,
        parentID: 0,
        date: '2016-05-01',
        name: 'wangxiaohu',
        hasChildren: true,
        address: 'No. 189, Grove St, Los Angeles',
    },
    {
        id: 4,
        parentID: 0,
        date: '2016-05-03',
        name: 'wangxiaohu',
        address: 'No. 189, Grove St, Los Angeles',
    },
])
let sortableObj = ref<Sortable>() // 拖动实例化对象
let relatedDom = ref<HTMLElement>() // 拖动到的dom
let dragID = ref() // 拖动DOM的数据ID

// 多维数组扁平化
function treetoarray(treeData: any[], childrenKey: string, callback?: Function): any[] {
    const tempTreeData = cloneDeep(treeData)
    let cacheData = []
    for (const iterator of tempTreeData) {
        if (callback) {
            cacheData.push(callback(iterator))
        } else {
            cacheData.push(iterator)
        }
        if (iterator[childrenKey]) {
            cacheData = cacheData.concat(treetoarray(iterator[childrenKey], childrenKey))
        }
    }
    return cacheData
}
// 一维数组组装
function arraytotree(onearray: User[]): User[] {
    let nodeMap: Map<number, User> = new Map()
    let tempArray: User[] = []
    for (const iterator of onearray) {
        iterator.children = []
        nodeMap.set(iterator.id, iterator)
    }
    console.log(nodeMap);

    for (const iterator of nodeMap.values()) {
        const nodeData = nodeMap.get(iterator.parentID)
        if (nodeData) {
            nodeData.children?.push(iterator)
            nodeData.hasChildren = true
        } else {
            tempArray.push(iterator)
        }
    }
    return tempArray
}

// 表格子任务加载
const subTaskLoad = (rowInfo: User, treeNode: any, resolve: (date: any[]) => void) => {
    const temp: User[] = [
        {
            id: 31,
            parentID: 3,
            date: '2016-05-01',
            name: 'wangxiaohu',
            address: 'No. 189, Grove St, Los Angeles',
        },
        {
            id: 32,
            parentID: 3,
            date: '2016-05-01',
            name: 'wangxiaohu',
            address: 'No. 189, Grove St, Los Angeles',
        },
    ]
    // 模拟接口请求
    setTimeout(() => {
        rowInfo.children = temp
        resolve(rowInfo.children || [])
    }, 1000)
}

// 表格行样式
const tableRowClassName = ({ row }: any) => {
    return `table-row drag-class-${row.id}`
}

// 清除拖动效果
const clearDragAnimation = () => {
    if (!relatedDom.value) {
        console.error('未获取到替换DOM')
        return
    }
    const tempRelated = relatedDom.value.querySelector('.drag-animation')
    tempRelated && relatedDom.value.removeChild(tempRelated)
    relatedDom.value.classList.remove('son-drag-animation')
}
// 表格拖动开始
const tableStart = (event: SortableEvent) => {
    // 在Sortable中onMove返回false时，拖动的那个元素不会触发onMove事件，所以手动添加上事件
    event.item.addEventListener(
        'dragover',
        useThrottleFn(() => {
            if (relatedDom.value) {
                clearDragAnimation()
                relatedDom.value = undefined
            }
        }, 300)
    )
}
// 表格拖动中
const tableMove = (evt: MoveEvent, originalEvent: Event) => {
    if (relatedDom.value && !evt.related.isEqualNode(relatedDom.value)) {
        // 如果替换的dom不一致，则删除原有的效果
        clearDragAnimation()
        relatedDom.value = evt.related
    } else if (!relatedDom.value) {
        relatedDom.value = evt.related
    }

    if ((originalEvent as DragEvent).offsetY > 2 && (originalEvent as DragEvent).offsetY <= 10) {
        clearDragAnimation()
        // 替换dom的前面
        const div = document.createElement('div')
        div.className = 'before drag-animation'
        evt.related.appendChild(div)
    } else if ((originalEvent as DragEvent).offsetY > 10 && (originalEvent as DragEvent).offsetY <= 20) {
        clearDragAnimation()
        // 替换dom的子级
        evt.related.classList.add('son-drag-animation')
    } else if ((originalEvent as DragEvent).offsetY > 20) {
        clearDragAnimation()
        // 替换dom的后面
        const div = document.createElement('div')
        div.className = 'after drag-animation'
        evt.related.appendChild(div)
    }
    return false
}
// 表格拖动结束
const tableEnd = () => {
    let tempRequest: { DragID: number; DropID: number; DropType: string } = {
        DragID: dragID.value,
        DropID: 0,
        DropType: '',
    }

    const oneArray = treetoarray(taskTableRef.value?.data || [], 'children')
    if (relatedDom.value) {
        const tempRelated = relatedDom.value.querySelector('.drag-animation')
        const isChild = relatedDom.value.className.includes('son-drag-animation')
        // 获取拖入方式
        if (tempRelated) {
            tempRequest.DropType = Array.from(tempRelated.classList).includes('before') ? 'before' : 'after'
        } else if (isChild) {
            tempRequest.DropType = 'inner'
        }
        // 获取拖入的ID
        relatedDom.value?.classList.forEach((item) => {
            if (item.includes('drag-class')) {
                tempRequest.DropID = Number(item.split('drag-class-')[1])
            }
        })

        // 获取拖动的下标
        const dragIndex = oneArray.findIndex((item: any) => item.id == tempRequest.DragID)
        const dragData = oneArray.find((item: any) => item.id === tempRequest.DragID)
        // 获取拖入的下标
        const dropIndex = oneArray.findIndex((item: any) => item.id == tempRequest.DropID)
        const dropData = oneArray.find((item: any) => item.id == tempRequest.DropID)
        if (dragIndex != -1 && dragData && dropIndex != -1 && dropData) {
            oneArray.splice(dragIndex, 1)
            switch (tempRequest.DropType) {
                case 'before':
                    oneArray.splice(dropIndex - 1, 0, dragData)
                    dragData.parentID = dropData.parentID
                    break;
                case 'after':
                    oneArray.splice(dropIndex, 0, dragData)
                    dragData.parentID = dropData.parentID
                    break;
                case 'inner':
                    oneArray.splice(dropIndex, 0, dragData)
                    dragData.parentID = dropData.id
                    break;
                default:
                    break;
            }
        }
        console.log(oneArray);
        tableData.value = []
        nextTick(() => {
            tableData.value = arraytotree(oneArray)
            console.log(taskTableRef.value!.store);

            // taskTableRef.value!.store.states.lazyTreeNodeMap.value![rowInfo?.ID || 0] = rowInfo?.SubNodes
            console.log(tableData.value);
            taskTableRef.value?.doLayout()
        })
        clearDragAnimation()
    }
}
// 设置拖动排序
const setDragSort = () => {
    // 获取容器元素
    const el = taskTableRef.value?.$el.querySelector('.el-table__body-wrapper tbody')
    if (!el) return
    sortableObj.value = new Sortable(el, {
        handle: '.drag-mark',
        forceFallback: false,
        onMove: tableMove,
        onStart: tableStart,
        onEnd: tableEnd,
    })
}

onMounted(() => {
    setDragSort()
})
</script>
<style scoped lang="scss">
:deep(.tree-table-drag) {
    .el-table td .cell {
        height: 32px;
    }

    .el-table td .cell,
    .el-table th .cell {
        position: relative;
        display: flex;
        align-items: center;
        font-size: 13px;

        .hidden-date-picker {
            width: 100%;
            visibility: hidden;
            position: absolute;
            top: 0px;
        }
    }

    .el-table [class*='el-table__row--level'] .el-table__expand-icon {
        height: 24px;
        width: 24px;
        margin-right: 0px;
    }

    .el-table__expand-icon>.el-icon {
        height: 24px;
        width: 24px;
    }

    .el-table .el-table__placeholder {
        width: 24px;
    }


    .table-row {
        cursor: pointer;
        position: relative;

        .show-button {
            display: none;
        }

        &:hover .show-button {
            display: initial;
        }
    }

    .drag-mark {
        padding: 10px;
        cursor: move;
    }
}

// 拖动效果
:deep(.drag-animation) {
    width: 100%;
    height: 0px;
    border-bottom: 2px solid #009688;
    position: absolute;
    left: 0px;
    z-index: 10;
    background-color: #009688;

    &::before {
        content: '';
        position: absolute;
        top: -4px;
        left: 0px;
        width: 10px;
        height: 10px;
        z-index: 10;
        border-radius: 50%;
        background-color: #009688;
    }
}

:deep(.before) {
    top: 0px;
}

:deep(.after) {
    bottom: 0px;
}

:deep(.son-drag-animation) {
    background-color: #009688;
    color: #ffffff;

    td {
        background-color: #009688 !important;
    }

    &:hover {
        color: #606266;
    }
}
</style>
