<template>
  <div class="upload-container">
    <div>
      <div class="input-btns" style="margin-bottom: 10px">
        <input ref="input" type="file" style="display: none" @change="handleChange" :multiple="multiple"
          :accept="accept ? accept : (img ? 'image/*' : (excel == true ? '.xls,.xlsx' : ''))" />
        <div v-if="img" class="upload-img">
          <!-- v-for="(file,index) in fileInfo.length>0?fileInfo: files" -->
          <div v-for="(file, index) in files" :key="index" class="img-item">
            <div class="operation">
              <div class="action">
                <i class="el-icon-view view" @click="previewImg(index)"></i>
                <i class="el-icon-delete remove" @click="removeFile(index)"></i>
              </div>
              <div class="mask"></div>
            </div>

            <img v-fallback :src="getImgSrc(file, index)" />
          </div>
          <div v-show="!autoUpload || (autoUpload && files.length < maxFile)" class="img-selector"
            :class="getSelector()">
            <div class="selector" @click="handleClick">
              <i class="el-icon-camera-solid"></i>
            </div>
            <div v-if="!autoUpload" class="s-btn" :class="{ readonly: changed }" @click="upload">
              <div>{{ loadText }}</div>
            </div>
          </div>
        </div>
        <el-button v-else @click="handleClick">选择{{ img ? '图片' : '文件' }}</el-button>

        <el-button v-if="!autoUpload && !img" type="info" :disabled="changed" @click="upload(true)"
          :loading="loadingStatus">上传文件</el-button>
      </div>
      <slot></slot>
      <div v-if="desc">
        <el-alert :title="getText() + '文件大小不超过' + (maxSize || 50) + 'M'" type="info" show-icon>
        </el-alert>
      </div>
      <slot name="content"></slot>
      <div v-if="!img">
        <ul class="upload-list" v-show="fileList">
          <li class="list-file" v-for="(file, index) in files" :key="index">
            <a>
              <span @click="fileOnClick(index, file)">
                <i :class="format(file)"></i>
                {{ file.name }}
              </span>
            </a>
            <span @click="removeFile(index)" class="file-remove">
              <i class="el-icon-close"></i>
            </span>
          </li>
        </ul>
      </div>
      <slot name="tip"></slot>
    </div>
    <vol-image-viewer ref="viewer"></vol-image-viewer>
  </div>
</template>
<script>
let OSS = {}// require('ali-oss');
import VolImageViewer from './VolImageViewer.vue';
export default {
  components: {
    'vol-image-viewer': VolImageViewer
  },
  props: {
    desc: {
      //是否显示默认介绍
      //是否多选
      type: Boolean,
      default: false
    },
    fileInfo: {
      //用于接收上传的文件，也可以加以默认值，显示已上传的文件，用户上传后会覆盖默认值
      type: Array,
      default: () => {
        return [];
      } //格式[{name:'1.jpg',path:'127.0.01/1.jpg'}]
    },
    downLoad: {
      //是否可以点击文件下载
      type: Boolean,
      default: true
    },
    multiple: {
      //是否多选
      type: Boolean,
      default: false
    },
    maxFile: {
      //最多可选文件数量，必须multiple=true，才会生效
      type: Number,
      default: 5
    },
    maxSize: {
      //文件限制大小3M
      type: Number,
      default: 50
    },

    autoUpload: {
      //选择文件后是否自动上传
      type: Boolean,
      default: true
    },
    img: {
      //图片类型  img>excel>fileTypes三种文件类型优先级
      type: Boolean,
      default: false
    },
    excel: {
      //excel文件
      type: Boolean,
      default: false
    },
    fileTypes: {
      //指定上传文件的类型
      type: Array,
      default: () => {
        return [];
      }
    },
    url: {
      //上传的url
      type: String,
      default: ''
    },
    uploadBefore: {
      //返回false会中止执行
      //上传前
      type: Function,
      default: (files) => {
        return true;
      }
    },
    uploadAfter: {
      //返回false会中止执行
      //上传后
      type: Function,
      default: (result, files) => {
        return true;
      }
    },
    onChange: {
      //选择文件时  //返回false会中止执行
      type: Function,
      default: (files) => {
        return true;
      }
    },
    // clear: {
    //   //上传完成后是否清空文件列表
    //   type: Boolean,
    //   default: true
    // },
    fileList: {
      //是否显示选择的文件列表
      type: Boolean,
      default: true
    },
    fileClick: {
      //点击文件事件
      type: Function,
      default: (index, file, files) => {
        return true;
      }
    },
    removeBefore: {
      //移除文件事件
      type: Function,
      default: (index, file, files) => {
        return true;
      }
    },
    append: {
      //此属性已废弃，多文件上传，默认追加文件
      type: Boolean,
      default: false
    },
    compress: {
      //开启图片压缩,后面根据需要再完善
      type: Boolean,
      default: true
    },
    compressMinSize: {
      //压缩的最小比例
      type: Number,
      default: 0.1
    },
    accept: {
      //接受的文件类型
      type: String,
      default: ''
    }
  },
  data() {
    return {
      changed: false, //手动上传成功后禁止重复上传，必须重新选择
      model: true,
      files: [],
      bigImg: '',
      imgTypes: ['gif', 'jpg', 'jpeg', 'png', 'bmp', 'webp', 'jfif'],
      loadingStatus: false,
      loadText: '上传文件'
    };
  },
  created() {
    //默认有图片的禁止上传操作
    if (this.fileInfo) {
      this.changed = true;
    }
    this.cloneFile(this.fileInfo);
  },
  watch: {
    fileInfo: {
      handler(files) {
        this.cloneFile(files);
      },
      deep: true
    }
  },
  methods: {
    cloneFile(files) {
      this.files = files.map((x) => {
        return {
          name: x.name || this.getFileName(x.path),
          path: x.path
        };
      });
    },
    getFileName(path) {
      if (!path) {
        return '未定义文件名';
      }
      let _index = path.lastIndexOf('/');
      return path.substring(_index + 1);
    },
    previewImg(index) {
      const imgs = this.files.map(x => { return this.getImgSrc(x) });
      this.$refs.viewer.show(imgs, index);
      //  this.base.previewImg(this.getImgSrc(this.files[index]));
      //  window.open(this.getImgSrc((this.files.length>0?this.files:this.fileInfo)[index]));
    },
    getSelector() {
      if (this.autoUpload) {
        return 'auto-selector';
      }
      return 'submit-selector';
    },
    getImgSrc(file, index) {
      if (file.hasOwnProperty('path')) {
        if (this.base.isUrl(file.path)) {
          return file.path;
        }
        //2020.12.27增加base64图片操作
        if (file.path.indexOf('/9j/') != -1) {
          return 'data:image/jpeg;base64,' + file.path;
        }
        if (file.path.substr(0, 1) == '/') {
          file.path = file.path.substr(1);
        }
        return this.http.ipAddress + file.path;
      }
      return window.URL.createObjectURL(file);
    },
    fileOnClick(index, file) {
      if (!this.fileClick(index, file, this.files)) {
        return;
      }
      //点击不下载
      if (!this.downLoad) {
        return;
      }
      if (!file.path) {
        this.$message.error('请先上传文件');
        return;
      }
      this.base.dowloadFile(
        file.path,
        file.name,
        {
          Authorization: this.$store.getters.getToken()
        },
        this.http.ipAddress
      );
    },
    getText() {
      if (this.img) {
        return '只能上传图片,';
      } else if (this.excel) {
        return '只能上传excel文件,';
      }
    },
    handleClick() {
      this.$refs.input.click();
    },
    handleChange(e) {
      //this.compress开启图片压缩,后面根据需要再完善
      // this.clearFiles();
      var result = this.checkFile(e.target.files);
      if (!result) {
        return;
      }

      this.changed = false;
      //如果传入了FileInfo需要自行处理移除FileInfo
      if (!this.onChange(e.target.files)) {
        return;
      }
      for (let index = 0; index < e.target.files.length; index++) {
        const element = e.target.files[index];
        element.input = true;
      }
      if (!this.multiple) {
        this.files.splice(0);
      }
      this.files.push(...e.target.files);

      this.$refs.input.value = null;
      if (this.autoUpload && result) {
        this.upload(false);
      }
    },
    removeFile(index) {
      //如果传入了FileInfo需要自行处理移除FileInfo
      //t移除文件
      let removeFile = this.files[index];
      //删除的还没上传的文件
      if (removeFile.input) {
        this.files.splice(index, 1);
      } else {
        this.fileInfo.splice(index, 1);
      }
      if (!this.removeBefore(index, removeFile, this.fileInfo)) {
        return;
      }
    },
    clearFiles() {
      this.files.splice(0);
    },
    getFiles() {
      return this.files;
    },
    convertToFile(dataurl, filename) {
      let arr = dataurl.split(',');
      let mime = arr[0].match(/:(.*?);/)[1];
      let suffix = mime.split('/')[1];
      let bstr = atob(arr[1]);
      let n = bstr.length;
      let u8arr = new Uint8Array(n);
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      // new File返回File对象 第一个参数是 ArraryBuffer 或 Bolb 或Arrary 第二个参数是文件名
      // 第三个参数是 要放到文件中的内容的 MIME 类型
      return new File([u8arr], `${filename}.${suffix}`, {
        type: mime,
        input: true
      });
    },
    async compressImg(file) {
      let fileSize = file.size / 1024 / 1024;
      let read = new FileReader();
      read.readAsDataURL(file);
      return new Promise((resolve, reject) => {
        read.onload = (e) => {
          let img = new Image();
          img.src = e.target.result;
          let _this = this;
          img.onload = function () {
            //默认按比例压缩
            let w = this.width;
            let h = this.height;
            let canvas = document.createElement('canvas');
            let ctx = canvas.getContext('2d');
            canvas.setAttribute('width', w);
            canvas.setAttribute('height', h);
            ctx.drawImage(this, 0, 0, w, h);
            let rate = 0.3;
            if (fileSize > 2) {
              rate = 0.1;
            } else if (fileSize > 1) {
              rate = 0.1;
            }
            if (_this.compressMinSize > rate) {
              rate = _this.compressMinSize;
            }
            // rate=1;
            let base64 = canvas.toDataURL('image/jpeg', rate);
            resolve(_this.convertToFile(base64, file.name));
          };
        };
      });
    },
    async uploadOSS() {
      await this.http.get('api/alioss/getAccessToken', {}, false).then(async (x) => {
        if (!x.status) return this.$Message.error(x.message);
        let client = new OSS({
          // yourRegion填写Bucket所在地域。以华东1（杭州）为例，Region填写为oss-cn-hangzhou。
          region: x.data.region,
          // 从STS服务获取的临时访问密钥（AccessKey ID和AccessKey Secret）。
          accessKeyId: x.data.accessKeyId,
          accessKeySecret: x.data.accessKeySecret,
          // 从STS服务获取的安全令牌（SecurityToken）。
          stsToken: x.data.securityToken,
          // 填写Bucket名称。
          bucket: x.data.bucket
        });
        console.log(this.files);
        for (let index = 0; index < this.files.length; index++) {
          const file = this.files[index];
          if (file.input) {
            let result = await client.put(
              x.data.bucketFolder + '/' + x.data.unique + file.name,
              file
            );
            // 如果有配置cdn，返回的url需要拼接cdn
            if (window.oss.ali.cdn) {
              result.url = new URL(x.data.bucketFolder + '/' + x.data.unique + file.name, window.oss.ali.cdn).toString();
            }
            file.path = result.url;
            file.newName = x.data.unique + file.name;
          }
        }

        this.fileInfo.splice(0);
        // }
        let _files = this.files.map((file) => {
          return {
            name: file.newName || file.name,
            path: file.path
          };
        });
        this.fileInfo.push(..._files);
        //2021.09.25修复文件上传后不能同时下载的问题
        this.files = _files;
      });
      return;
    },
    async upload(vail) {
      if (vail && !this.checkFile()) return false;
      if (!this.url) {
        return this.$message.error('没有配置好Url');
      }
      if (!this.files || this.files.length == 0) {
        return this.$message.error('请选择文件');
      }
      //增加上传时自定义参数，后台使用获取Utilities.HttpContext.Current.Request.Query["字段"]
      let params = {};
      if (!this.uploadBefore(this.files, params)) {
        return;
      }
      let paramText = "";
      if (Object.keys(params).length) {
        paramText = "?1=1";
        for (const key in params) {
          let value = params[key];
          if (typeof (value) == 'object') {
            value = JSON.stringify(value)
          }
          paramText += `&${key}=${value}`
        }
      }

      this.loadingStatus = true;
      this.loadText = '上传中..';
      if (window.oss && window.oss.ali.use) {
        await this.uploadOSS();
        this.loadingStatus = false;
        this.loadText = '上传文件';
        if (!this.uploadAfter({ status: true }, this.fileInfo, this.files)) {
          this.changed = false;
          return;
        } else {
          this.changed = true;
        }
        this.$message.success('上传成功');
        return;
      }

      var forms = new FormData();
      for (let index = 0; index < this.files.length; index++) {
        let file = this.files[index];
        if (file.input) {
          //2023.07屏蔽图片压缩
          // let name = file.name.split('.');
          // name = name[name.length - 1].toLocaleLowerCase();
          // let isImg = this.imgTypes.indexOf(name) != -1;
          // if (isImg && (name == 'jpg' || name == 'jpeg')) {
          //   //>200KB的开启压缩
          //   if (isImg && file.size / 1024 / 1024 > 0.2) {
          //     console.log('压缩前' + file.size);
          //     file = await this.compressImg(file);
          //     file.compress = true;
          //     this.files[index] = file;
          //     this.files[index].input = true;
          //     console.log('压缩后' + file.size);
          //   }
          // }
          forms.append('fileInput', file, file.name);
        }
      }
      // forms.append("fileInput", this.files);

      this.http
        .post(this.url + paramText, forms, this.autoUpload ? '正在上传文件' : '',
          //高版本axios这里必须要指定header
          {
            headers: { 'Content-Type': 'multipart/form-data' }
          })
        .then(
          (x) => {
            // this.$refs.uploadFile.clearFiles();
            this.loadingStatus = false;
            this.loadText = '上传文件';
            if (!this.uploadAfter(x, this.files)) {
              this.changed = false;
              return;
            } else {
              this.changed = true;
            }
            this.$message.success(x.message);
            this.changed = x.status;
            if (!x.status) {
              // this.files = null;
              return;
            }
            //单选清除以前的数据
            //  if (!this.multiple) {
            this.fileInfo.splice(0);
            // }
            let _files;
            if (this.multiple && this.base.isUrl(x.data.split(',')[0])) {
              _files = this.files.filter((file) => { return file.path });
              _files.push(...x.data.split(',').map(x => {
                return {
                  name: x.split('/').pop(),
                  path: x
                }
              }))
            } else {
              _files = this.files.map((file) => {
                if (file.path) {
                  return file;
                }
                return {
                  name: file.name,
                  path: file.path || (this.base.isUrl(x.data) ? x.data : (x.data + file.name))
                };
              });
            }
            this.fileInfo.push(..._files);
            //2021.09.25修复文件上传后不能同时下载的问题
            this.files = _files;
          },
          (error) => {
            this.loadText = '上传文件';
            this.loadingStatus = false;
          }
        );
    },
    format(file, checkFileType) {
      const format =
        file.name
          .split('.')
          .pop()
          .toLocaleLowerCase() || '';
      let fileIcon = 'el-icon-document';
      if (this.fileTypes.length > 0 && checkFileType != undefined) {
        if (this.fileTypes.indexOf(format) != -1) {
          return true;
        }
        return false;
      }
      if (
        checkFileType &&
        !(checkFileType instanceof Array) &&
        checkFileType != 'img' &&
        checkFileType != 'excel'
      ) {
        if (checkFileType.indexOf(format) > -1) {
          return true;
        } else {
          return false;
        }
      }

      if (checkFileType == 'img' || this.imgTypes.indexOf(format) > -1) {
        if (checkFileType == 'img') {
          if (this.imgTypes.indexOf(format) > -1) {
            return true;
          } else {
            return false;
          }
        }
        fileIcon = 'el-icon-picture-outline';
      }
      if (
        ['mp4', 'm3u8', 'rmvb', 'avi', 'swf', '3gp', 'mkv', 'flv'].indexOf(
          format
        ) > -1
      ) {
        fileIcon = 'el-icon-document';
      }
      if (['mp3', 'wav', 'wma', 'ogg', 'aac', 'flac'].indexOf(format) > -1) {
        fileIcon = 'el-icon-document';
      }
      if (['doc', 'txt', 'docx', 'pages', 'epub', 'pdf'].indexOf(format) > -1) {
        fileIcon = 'el-icon-document';
      }
      if (
        checkFileType == 'excel' ||
        ['numbers', 'csv', 'xls', 'xlsx'].indexOf(format) > -1
      ) {
        if (checkFileType == 'excel') {
          if (['numbers', 'csv', 'xls', 'xlsx'].indexOf(format) > -1) {
            return true;
          } else {
            return false;
          }
        }
        fileIcon = 'el-icon-document';
      }
      return fileIcon;
    },
    beforeUpload() { },
    checkFile(inputFiles) {
      const files = this.files;

      if (
        this.multiple &&
        files.length + (inputFiles || []).length > (this.maxFile || 5)
      ) {
        this.$message.error(
          '最多只能选【' +
          (this.maxFile || 5) +
          '】' +
          (this.img ? '张图片' : '个文件') +
          ''
        );
        return false;
      }
      if (!inputFiles) {
        inputFiles = this.files.filter((x) => {
          return x.input;
        });
      }
      let names = [];
      for (let index = 0; index < inputFiles.length; index++) {
        const file = inputFiles[index];
        if (names.indexOf(file.name) != -1) {
          file.name = '(' + index + ')' + file.name;
        }
        names.push(file.name);
        if (this.img && !this.format(file, 'img')) {
          this.$message.error('选择的文件【' + file.name + '】只能是图片格式');
          return false;
        }
        if (this.excel && !this.format(file, 'excel')) {
          this.$message.error('选择的文件【' + file.name + '】只能是excel文件');
          return false;
        }
        if (
          this.fileTypes &&
          this.fileTypes.length > 0 &&
          !this.format(file, this.fileTypes)
        ) {
          this.$message.error(
            '选择的文件【' +
            file.name +
            '】只能是【' +
            this.fileTypes.join(',') +
            '】格式'
          );
          return false;
        }
        if (file.size > (this.maxSize || 50) * 1024 * 1024) {
          this.$message.error(
            '选择的文件【' +
            file.name +
            '】不能超过:' +
            (this.maxSize || 50) +
            'M'
          );
          return false;
        }
      }
      return true;
    }
  }
};
</script>
<style lang="less" scoped>
.upload-list {
  padding-left: 0;
  list-style: none;

  .list-file {
    line-height: 20px;
    padding: 4px;
    color: #515a6e;
    border-radius: 4px;
    transition: background-color 0.2s ease-in-out;
    overflow: hidden;
    position: relative;

    font-size: 13px;

    .file-remove {
      display: none;
      right: 0;
      //  margin-left: 50px;
      color: #0e9286;
    }
  }

  .list-file:hover {
    cursor: pointer;

    .file-remove {
      display: initial;
    }

    color: #2d8cf0;
  }
}

.upload-container {
  display: inline-block;
  width: 100%;
  // padding: 10px;

  // min-height: 250px;
  border-radius: 5px;

  .alert {
    margin-top: 43px;
  }

  .button-group>* {
    float: left;
    margin-right: 10px;
  }

  .file-info>span {
    margin-right: 20px;
  }
}

.upload-img {
  display: inline-block;

  .img-item:hover .operation {
    display: block;
  }

  .img-item,
  .img-selector {
    position: relative;
    cursor: pointer;
    margin: 0 10px 10px 0;
    float: left;
    width: 65px;
    height: 65px;
    border: 1px solid #c7c7c7;
    overflow: hidden;
    border-radius: 5px;
    box-sizing: content-box;

    img {
      margin: 0;
      padding: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
    }

    .operation {
      display: none;
      position: absolute;
      top: 0;
      bottom: 0;
      left: 0;
      right: 0;

      .action {
        opacity: 0.6;
        text-align: center;
        background: #151515de;
        font-size: 14px;
        position: absolute;
        z-index: 90;
        width: 100%;
        bottom: 3px;
        bottom: 0;
        color: #ded5d5;
        padding-right: 7px;
        padding-bottom: 3px;
        line-height: 20px;

        .el-icon-view {
          margin: 0 10px;
        }
      }

      .mask {
        opacity: 0.6;
        background: #9e9e9e;
        top: 0;
        width: 100%;
        height: 100%;
        position: absolute;
      }
    }
  }

  .img-selector {
    font-size: 50px;
    text-align: center;

    i {
      position: relative;
      font-size: 40px;
      color: #6f6f6f;
    }
  }

  .auto-selector {
    .selector {
      line-height: 64px;
    }
  }

  .selector {
    color: #a0a0a0;
  }

  .submit-selector {
    .s-btn {
      line-height: 22px;
      font-size: 12px;
      top: -6px;
      // padding: 2px;
      position: relative;
      background: #2db7f5;
      color: white;
    }

    .selector {
      line-height: 50px;
    }

    .readonly {
      background: #8c8c8c;
    }
  }
}

.big-model {
  width: 100%;
  height: 100%;
  position: relative;

  .m-img {}

  .mask {
    position: absolute;
    opacity: 0.6;
    background: #eee;
    top: 0;
    width: 100%;
    height: 100%;
    position: absolute;
  }
}

.auto-upload {
  z-index: 9999999;
  width: 100%;
  height: 100%;
  position: fixed;
  top: 0;
  left: 0;

  .j-content {
    text-align: center;
    font-size: 17px;
    top: 40%;
    position: absolute;
    z-index: 999;
    left: 0;
    right: 0;
    width: 240px;
    /* height: 100%; */
    margin: auto;
    background: white;
    /* bottom: 30px; */
    line-height: 50px;
    border-radius: 6px;
    border: 1px solid #d2d2d2;
  }

  .mask {
    cursor: pointer;
    opacity: 0.6;
    width: 100%;
    height: 100%;
    background: #101010;
  }
}
</style>
