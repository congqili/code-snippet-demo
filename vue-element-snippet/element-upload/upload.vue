/* eslint-disable require-atomic-updates */
<template>
  <div class="g-upload" :style="styles">
    <section v-if="uploadBtn">
      <el-upload
        ref="upload"
        v-loading="upFlag"
        action=""
        :http-request="handleUploadYY"
        class="upload-file"
        :list-type="listType"
        :before-upload="beforeUpload"
      >
        <el-button
          ref="uploadBtn"
          size="small"
          :type="'info'"
          :icon="btnIcon"
          :loading="uploadLoading"
        >
          {{ btnText }}
        </el-button>
      </el-upload>
      <slot v-if="uploadBtn" name="tip" />
      <slot name="btn-right" />
    </section>
  
    <div
      v-if="fileList.length && !textFlag"
      :class="['file-list']"
    >
      <template v-for="(item, index) in fileList">
        <div :key="index" class="file-item">
          <div class="left">
            <i class="el-icon-paperclip"></i>
          </div>

          <div class="center">
            <div class="card-tit">
                <div class="ct-name">{{item.name}}</div>
                <div class="ct-size">
                  {{ item.fileSize }}
                </div>
              </div>
            <div>
              <el-link
                v-if="downloadBtn"
                v-loading="item.upLoading"
                class="btn-down"
                type="primary"
                @click="downLoadFile(item)"
              >
                下载
              </el-link>
              <el-link
                v-if="removeBtn"
                v-loading="item.deLoading || false"
                type="danger"
                @click="delFile(item, index)"
              >
                删除
              </el-link>
            </div>
          </div>
        </div>
      </template>
    </div>
  </div>
</template>

<script>
import formixs from '@/mixins/form';
import request from '@/utils/request';
import { guid, downBlobFolder } from '@/utils/utils';
import { downloadSliceFile } from '@/api/Common'
export default {
  name: 'UpFolder',
  mixins: [formixs],
  props: {
    // itemId
    itemId: {
      type: String,
      default: ''
    },
    type: {
      type: String,
      default: ''
    },
    textFlag: {
      type: Boolean,
      default: false
    },
    disabled: {
      type: Boolean,
      default: false
    },
    // 上传地址
    upPath: {
      type: String,
      default: ''
    },
    // 下载地址
    downPath: {
      type: String,
      default: ''
    },
    // 删除地址
    delPath: {
      type: String,
      default: ''
    },
    // 按钮信息
    btnText: {
      type: String,
      default: '点击上传'
    },
    // 上传图标信息
    btnIcon: {
      type: String,
      default: 'el-icon-upload'
    },
    // 上传类型
    listType: {
      type: String,
      default: 'picture'
    },
    // 分类编码 (同一业务下多种分类文件)
    classifyCode: {
      type: String,
      default: ''
    },
    // 绑定的上传文件信息  如下示例
    value: {
      type: Array,
      default: () => {
        return [];
      }
    },
    // [{ fileId: 34, fileName: '54607d1c8c87e.jpg',flag: false,index: 1,size: 1321754 }]
    // 是否显示下载按钮
    downloadBtn: {
      type: Boolean,
      default: true
    },
    // 是否显示删除按钮
    removeBtn: {
      type: Boolean,
      default: true
    },
    // 是否显示上传按钮
    uploadBtn: {
      type: Boolean,
      default: true
    },
    // 文件最大值（MB）
    maxSize: {
      type: Number,
      default: 200
    },
    // 文件最小值（KB）
    minSize: {
      type: Number,
      default: 0
    },
    // 上传信息部分和规范提示
    waryMsg: {
      type: String,
      default: ''
    },
    // 上传个数
    limit: {
      type: Number,
      default: 10
    },
    /*
      上传类型
      不传 -不- 校验类型
    */
    loadType: {
      type: Array,
      default: () => {
        return [];
      }
    },
    hasAlone: {
      type: Boolean,
      default: false
    },
    // 上传按钮样式
    btnType: {
      type: String,
      default: ''
    },
    //  显示大小
    listSize: {
      type: String,
      default: ''
    },
    // 删除文件，单独调用
    remark: {
      type: Boolean,
      default: false
    },
    // 上传类型  data 数据 file  存储数据库
    sceneType: {
      type: String,
      default: 'file'
    },
    // 样式
    styles: {
      type: String,
      default: ''
    }
  },
  data() {
    return {
      // 上传信息集合
      fileList: this.value,
      uploadLoading: false,
      // 文件
      file: '',
      // 文件分片  以5MB去分片
      shardSize: 5 * 1024 * 1024,
      // 分片索引
      shardIndex: 1,
      // 文件大小
      size: '',
      // 总片数
      shardTotal: '',
      // 文件后缀
      suffix: '',
      // 文件名不带后缀
      bName: '',
      upLoading: false,
      deLoading: false,
      compressFile: ['rar', 'zip', 'arj', 'z', '7z'],
      fileType: {
        word: ['doc', 'docx'],
        excel: ['xls', 'xlsx', 'csv', 'et'],
        ppt: ['ppt', 'pptx']
      },
      uuid: null,
      upFlag: false,
      imgsArr: [], // 分片下载blob集合
      downsShardSize: 0// 下载分数量
    };
  },
  computed: {
    // 文件等于10，按钮置灰
    disUpBtn() {
      const flag = this.fileList.length > this.limit - 1;
      return flag;
    },
    // 当前模块名
    modelName() {
      const routeQuery = this.$route
      const { name, meta } = routeQuery
      const routeName = meta?.title || name
      return routeName.replace('详情', '')
    }
  },
  watch: {
    value: {
      handler(val) {
        this.fileList = val || [];
      },
      deep: true
    },
    upFlag: {
      handler(val) {
        this.$emit('upFlag', val);
      },
      deep: true
    }
  },

  methods: {
    // 设置文件名
    setFileName(val) {
      if (val?.suffixName) {
        return `${val.fileName}.${val.suffixName}`;
      } else {
        return `${val.fileName}`;
      }
    },
    // 触发上传事件
    handleUploadYY(e) {
      this.uuid = guid();
      this.file = e.file;
      const { name, size } = e.file;
      this.name = name;
      this.size = size;
      // 总片数
      this.shardTotal = Math.ceil(size / this.shardSize);
      // 文件后缀
      this.suffix = name.substring(name.lastIndexOf('.') + 1, name.length).toLowerCase();
      this.bName = name.substring(0, name.lastIndexOf('.')).toLowerCase();
      this.upFlag = true;
      if (this.sceneType === 'file') {
        this.uploadInsertFile(1);
      } else {
        var fd = new FormData();
        fd.append('fileName', e.file.name);
        fd.append('file', e.file);
        fd.append('name', e.file.name);
        fd.append('classifyCode', e.file.classifyCode);
        this.uploadRequey(fd)
      }
    },
    // 上穿文件方法
    uploadInsertFile(shardIndex) {
      const {
        file,
        shardSize,
        bName,
        size,
        suffix,
        shardTotal,
        classifyCode,
        uuid
      } = this;
      var fd = new FormData();
      // 定义分片的起始位置
      var start = (shardIndex - 1) * shardSize;
      // 定义分片结束的位置
      var end = Math.min(file.size, start + shardSize);
      // 从文件中截取当前的分片数据
      var fileShard = file.slice(start, end);
      const fileName = bName.replace(/\s*/g, '');
      fd.append('file', fileShard);
      fd.append('fileName', fileName);
      fd.append('fileSize', size);
      fd.append('seqNo', shardIndex);
      fd.append('splitNum', shardTotal);
      fd.append('suffixName', suffix);
      fd.append('uuid', uuid);
      if (classifyCode) {
        fd.append('classifyCode', classifyCode);
      }

      // 这里添加判断，分片的最后一次
      request(
        {
          url: this.upPath,
          method: 'POST',
          data: fd,
          uploadModuleName: shardIndex === shardTotal ? this.modelName : null
        }
      )
        .then((res) => {
          const { code, context } = res;
          if (code !== 'K-000000') return;
          if (context?.flag && shardIndex < shardTotal) {
            var index = shardIndex + 1;
            this.uploadInsertFile(index);
          } else {
            this.upFlag = false;
            this.fileList.length > 0
              ? (this.fileList = [...this.fileList, context])
              : (this.fileList = [context]);
            this.$message.success('上传成功');
            this.$emit('input', this.fileList);
            this.$emit('getFilesAllName', {
              file,
              shardSize,
              bName,
              size,
              suffix,
              shardTotal,
              classifyCode,
              uuid,
              context
            })
          }
        })
        .catch(() => {
          this.upFlag = false;
        });
    },
    // 上传后端读取数据方式
    uploadRequey(option) {
      request({
        url: this.upPath,
        method: 'POST',
        data: option
      })
        .then((res) => {
          const { code, context } = res;
          if (code !== 'K-000000') return;
          this.upFlag = false;
          this.$message.success('上传成功');
          this.$emit('input', context); //  上传成功后，添加回调
        })
        .catch(() => {
          this.upFlag = false;
        });
    },
    // 下载文件
    async downLoadFile(item) {
      this.$set(item, 'upLoading', true);
      const downsShardSize = Math.ceil(+item.fileSize / this.shardSize)
      const fileId = item.fileId
      const initNum = 1
      this.imgsArr = []
      this.pollFiles(item, fileId, downsShardSize, initNum)
    },
    /*
      轮询分片下载
      item：当前对象
      id:文件fileId
      size:下载总片数
      num：当前片数
    */
    async pollFiles(item, id, size, num) {
      const response = await downloadSliceFile(
        {
          fileId: id,
          seqNo: num
        },
        {
          downLoadModuleName: num === size ? this.modelName : null
        }
      )
      if (!response?.data) return this.$message.error('下载失败')

      this.imgsArr.push(response.data)// 分片文件集合

      num = num + 1

      if (num <= size) {
        this.pollFiles(item, id, size, num)
      } else {
        const headName = response.headers?.filename || ''
        const data = this.imgsArr
        downBlobFolder(data, headName)
        item.upLoading = false;
      }
    },
    // 删除文件
    async delFile(item, index) {
      if (!this.remark) {
        this.$set(item, 'deLoading', true);
        request({
          url: this.delPath,
          method: 'POST',
          headers: {
            'business_id': item.fileId
          },
          deleteFileModuleName: this.modelName,
          delete_title: {
            title: encodeURIComponent('删除文件'),
            value: encodeURIComponent(item.fileName)
          },
          data: {
            fileId: item.fileId
          }
        }).then(() => {
          item.deLoading = false;
          this.fileList.splice(index, 1);
          this.$emit('input', this.fileList);
          this.$message.success('删除成功');
        });
      } else {
        this.$emit('delFiles', item, index)
      }
    },
    // 文件上传校验
    beforeUpload(file) {
      const { name, size } = file;
      const {
        maxSize,
        minSize,
        limit,
        loadType,
        compressFile,
        hasAlone,
        fileType
      } = this;
      // 添加worl，excel文件后缀的类型
      const loadTypeArr = [];
      const needReplace = []

      Object.keys(fileType).forEach(item => {
        needReplace.push(item)
      })
      loadType.forEach(item => {
        if (needReplace.includes(item)) {
          loadTypeArr.push(...fileType[item])
        } else {
          loadTypeArr.push(item)
        }
      })
      const type = name.slice(name.lastIndexOf('.') + 1);
      const isType = loadTypeArr.includes(type.toLowerCase());
      const isCompress = compressFile.includes(type.toLowerCase());
      // 校验压缩文件
      if (isCompress) {
        this.$message.warning(`不可以上传压缩文件！`);
        return false;
      }
      // 校验文件类型
      if (!isType && loadType.length > 0) {
        this.$message.warning(`请上传${loadType.join('、')}格式的文件！`);
        return false;
      }
      // 文件大小为0
      if (size === 0) {
        this.$message.warning(`文件为空！`);
        return false;
      }
      // 校验文件数量
      if (limit > 0 && this.fileList.length >= limit) {
        this.$message.warning(`上传附件数量不能超过 ${limit} 个！`);
        return false;
      }
      // 特殊情况 单独校验图片
      if (hasAlone) {
        const hasAlone = [
          'bmp',
          'jpeg',
          'gif',
          'psd',
          'png',
          'tiff',
          'tga',
          'eps',
          'jpg'
        ].includes(type.toLowerCase());
        if (hasAlone && ((size / 1024 <= 200) || (size / 1024 / 1024 > 2))) {
          this.$message.warning('上传图片大小须为200KB-2M！');
          return false;
        }
      }
      // 校验文件最大，最小值
      const isLtMaxM = size / 1024 / 1024 < maxSize;
      const isLtMinM = size / 1024 > minSize;
      if (!isLtMaxM || !isLtMinM) {
        // const msg = this.waryMsg ? this.waryMsg : `上传文件大小不能超过 ${maxSize}MB${minSize > 0 ? '，小于' + minSize + 'KB' : ''}！`
        this.$message.warning('文件大小超出限制！');
        return false;
      }
      return true;
    }

  }
};
</script>

<style lang="scss" scoped>
.upload-file {
  display: inline-block;
  ::v-deep {
    .el-upload {
      text-align: left;
    }
    .el-upload-list {
      display: none;
    }
  }
}

.file-list.small {
  .file-item {
    padding: 5px;
    .left {
      width: 20px;
      height: 20px;
      font-size: 14px;
      min-width: 20px;
      line-height: 20px;
    }
    .center {
      .btn-down {
        margin-right: 10px;
      }
      .ct-size {
        display: none;
      }
    }
  }
}
.file-list {
  border-top: 1px solid #ebeef5;
  display: flex;
  margin-top: 15px;
  padding-top: 15px;
  line-height: 1.4;
  flex-wrap: wrap;
  .file-item {
    width: 215px;
    display: flex;
    background-color: #fff;
    border-radius: 10px;
    box-shadow: 0 0 7px 0px rgba(#000, 0.1);
    padding: 12px;
    align-items: center;
    margin-right: 20px;
    margin-bottom: 20px;

    .left {
      width: 36px;
      min-width: 36px;
      height: 36px;
      line-height: 36px;
      border-radius: 50%;
      background-color: #419efe;
      color: #fff;
      font-size: 28px;
      text-align: center;
    }

    .center {
      margin: 0 10px;
      flex: 1;
      max-width: 80px;
    }

    .right {
      align-self: start;
      color: #a7a7a7;
      font-size: 12px;
    }
  }
}

.upload-btn {
  &.is-text {
    color: #409eff;
  }
}
.card-tit {
  position: relative;
  .ct-name {
    width: 70px;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
  }
  .ct-size {
    position: absolute;
    right: -70px;
    top: 0;
    font-size: 14px;
    color: #999999;
  }
}
.btn-down {
  margin-right: 20px;
}
.hindeTB {
  border: none;
}
.assembly {
  color: #409eff;
  min-width: 75px;
  height: 33px;
  font-size: 12px;
  &.el-button {
    background-color: #fff;
    border-color: #fff;
    &:hover,
    &:focus {
      background-color: #fff;
      border-color: #fff;
      color: #409eff;
    }
  }
}
.assemblyList {
  margin: 0;
  padding: 0;
  ::v-deep {
    .file-item {
      box-shadow: none;
      margin: 10px 0;
    }
  }
}
.blueText{
    background: none;
    border: none;
    color: #409EFF!important;
    &:hover{
          background: none;
    border: none;
    color: #409EFF!important;

    }
    &:focus{
                background: none;
    border: none;
    color: #409EFF!important;
    }
}
</style>
