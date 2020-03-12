<template>
  <ebook-dialog :title="title" ref="dialog">
    <!--替换slot插槽1-->
    <!--移动书籍弹框-->
    <div class="dialog-list-wrapper" v-if="!ifNewGroup">
      <div class="dialog-list-item" :class="{'is-add': item.edit  ? item.edit === 1 : false}"
           v-for="(item, index) in categoryList" :key="index"
           @click="onGroupClick(item)"
           v-if="(item.edit === 2 && isInGroup) || item.edit !== 2 || !item.edit">
        <!--各分组名-->
        <div class="dialog-list-item-text">{{item.title}}</div>
        <!--在选择移动分组时,自身所在的分组会被打勾-->
        <div class="dialog-list-icon-wrapper" v-if="isInGroup && shelfCategory.id === item.id">
          <!--图标: √-->
          <span class="icon-check"></span>
        </div>
      </div>
    </div>
    <!--新建分组的弹框-->
    <div class="dialog-new-group-wrapper" v-else>
      <div class="dialog-input-title-wrapper">
        <!--标题: 分组名-->
        <span class="dialog-input-title">{{$t('shelf.groupName')}}</span>
      </div>
      <div class="dialog-input-wrapper">
        <div class="dialog-input-inner-wrapper">
          <input type="text" class="dialog-input" v-model="newGroupName" ref="dialogInput">
          <div class="dialog-input-clear-wrapper" @click="clear" v-show="newGroupName && newGroupName.length > 0">
            <!--输入框中的清楚按钮 clear-->
            <span class="icon-close-circle-fill"></span>
          </div>
        </div>
      </div>
    </div>
    <!--替换slot插槽2-->
    <div slot="btn" class="group-dialog-btn-wrapper">
      <!--取消按钮-->
      <div class="dialog-btn" @click="hide">{{$t('shelf.cancel')}}</div>
      <!--在新建分组的弹框中才会显示-->
      <div class="dialog-btn" @click="createNewGroup"
           :class="{'is-empty': newGroupName && newGroupName.length === 0}"
           v-if="ifNewGroup">
        <!--确认按钮-->
        {{$t('shelf.confirm')}}
      </div>
    </div>
  </ebook-dialog>
</template>

<script>
  import EbookDialog from '../common/Dialog'
  import { storeShelfMixin } from '../../utils/mixin'
  import { removeAddFromShelf, appendAddToShelf } from '../../utils/store'
  import { saveBookShelf } from '../../utils/localStorage'

  export default {
    name: 'group-dialog',
    mixins: [storeShelfMixin],
    components: {
      EbookDialog
    },
    props: {
      showNewGroup: {
        type: Boolean,
        default: false
      },
      groupName: String
    },
    computed: {
      isInGroup() {
        return this.currentType === 2
      },
      // 分组弹框中的默认按钮
      defaultCategory() {
        return [
          {
            // 新建分组
            title: this.$t('shelf.newGroup'),
            edit: 1
          },
          {
            // 移出分组
            title: this.$t('shelf.groupOut'),
            edit: 2
          }
        ]
      },
      category() {
        return this.shelfList.filter(item => item.type === 2)
      },
      categoryList() {
        return [...this.defaultCategory, ...this.category]
      },
      title() {
        return !this.ifNewGroup ? this.$t('shelf.moveBook') : this.$t('shelf.newGroup')
      }
    },
    data() {
      return {
        ifNewGroup: false,
        newGroupName: ''
      }
    },
    methods: {
      show() {
        this.ifNewGroup = this.showNewGroup
        this.newGroupName = this.groupName
        this.$refs.dialog.show()
      },
      hide() {
        // 隐藏dialog
        this.$refs.dialog.hide()
        // 在200ms之后才执行,因为hide要持续200ms
        setTimeout(() => {
          // 关闭新增分组的弹窗
          this.ifNewGroup = false
        }, 200)
      },
      onGroupClick(item) {
        if (item.edit && item.edit === 1) {
          // 点击: 新建分组
          this.ifNewGroup = true
        } else if (item.edit && item.edit === 2) {
          // 点击: 移出分组
          this.moveOutFromGroup(item)
        } else {
          // 点击: 移动到分组
          this.moveToGroup(item)
        }
      },
      clear() {
        this.newGroupName = ''
      },
      moveToGroup(group) {
        this.setShelfList(
                // 遍历书架上的每一本书,保留没有被选中的书籍
                this.shelfList.filter(book => {
                  // book.itemList代表一个分组
                  if (book.itemList) {
                    // 对这个分组中进行遍历,保留没有被选中的书籍(this.shelfSelected.indexOf(subBook) < 0)
                    book.itemList = book.itemList.filter(subBook => this.shelfSelected.indexOf(subBook) < 0)
                  }
                  return this.shelfSelected.indexOf(book) < 0
                }))
                // 得到一个结果,然后:
                .then(() => {
                  if (group && group.itemList) {
                    // group是我们要所要移动的分组
                    // 将被选中的书籍和分组中的书籍进行合并
                    group.itemList = [...group.itemList, ...this.shelfSelected]
                  }
                  // 对这个分组中的每一本书的id进行重新计算
                  group.itemList.forEach((item, index) => {
                    item.id = index + 1
                  })
                  this.simpleToast(this.$t('shelf.moveBookInSuccess').replace('$1', group.title))
                  this.onComplete()
                })
      },
      moveOutFromGroup() {
        this.moveOutOfGroup(this.onComplete)
      },
      // 创建新的分组
      createNewGroup() {
        // 判断这个新分组的名称是否为空,不为空才进行操作
        if (!this.newGroupName || this.newGroupName.length === 0) {
          return
        }
        // 修改分组的名字
        if (this.showNewGroup) {
          this.shelfCategory.title = this.newGroupName
          this.onComplete()
        } else {
          // 建立一个新的分组对象
          const group = {
            // id 是书架当中倒数最后第二个元素的id + 1 ,最后一个元素是 添加书籍按钮
            id: this.shelfList[this.shelfList.length - 2].id + 1,
            itemList: [],
            selected: false,
            // 新的分组名
            title: this.newGroupName,
            // type=2 :表示是一个分组
            type: 2
          }
          // removeAddFromShelf(this.shelfList),先排除掉书架上的最后一个添加按钮
          let list = removeAddFromShelf(this.shelfList)
          // 然后把新的分组添加进来
          list.push(group)
          // 再把最后一个添加按钮加入回书架中
          list = appendAddToShelf(list)
          // 重新设置书架的值
          this.setShelfList(list).then(() => {
            // 然后把
            this.moveToGroup(group)
          })
        }
      },
      onComplete() {
        // 把书架保存起来
        saveBookShelf(this.shelfList)
        this.hide()
        this.setIsEditMode(false)
      }
    }
  }
</script>

<style lang="scss" rel="stylesheet/scss" scoped>
  @import "../../assets/styles/global";

  .dialog-list-wrapper {
    width: 100%;
    padding: 0 px2rem(20);
    box-sizing: border-box;
    font-size: px2rem(14);
    @include scroll;
    .dialog-list-item {
      display: flex;
      padding: px2rem(15) 0;
      box-sizing: border-box;
      color: #666;
      &.is-add {
        color: $color-blue;
        &:active {
          color: $color-blue-transparent;
        }
      }
      &:active {
        color: rgba(102, 102, 102, .5)
      }
      .dialog-list-item-text {
        flex: 1;
      }
      .dialog-list-icon-wrapper {
        flex: 0 0 px2rem(30);
        color: $color-blue;
        @include right;
      }
    }
  }

  .dialog-new-group-wrapper {
    width: 100%;
    padding: 0 px2rem(20);
    box-sizing: border-box;
    .dialog-input-title-wrapper {
      font-size: px2rem(10);
      margin-top: px2rem(20);
    }
    .dialog-input-wrapper {
      width: 100%;
      padding: 0 0 px2rem(30) 0;
      box-sizing: border-box;
      .dialog-input-inner-wrapper {
        display: flex;
        width: 100%;
        padding: px2rem(10) 0;
        box-sizing: border-box;
        border-bottom: px2rem(1) solid #eee;
        font-size: px2rem(14);
        color: #666;
        .dialog-input {
          flex: 1;
          border: none;
          &:focus {
            outline: none;
          }
        }
        .dialog-input-clear-wrapper {
          flex: 0 0 px2rem(30);
          color: #ccc;
          @include center;
          &:active {
            color: #999;
          }
        }
      }
    }
  }

  .group-dialog-btn-wrapper {
    width: 100%;
    @include center;
  }

  .dialog-btn {
    flex: 1;
    &.is-empty {
      color: rgba(255, 255, 255, .5);
    }
    &:active {
      color: rgba(255, 255, 255, .5)
    }
  }
</style>
