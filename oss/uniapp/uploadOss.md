# 文件上传到腾讯OSS

上传到腾讯OSS，需要获取腾讯云的配置信息，先在腾讯云开通COS服务，获取永久密钥和存储桶。

## uni-app 文件上传到腾讯OSS

uni-app 上传文件比web端上传文件要复杂一些，需要使用uni.uploadFile方法上传文件，上传文件时需要设置上传路径、上传文件名、上传文件模块等信息。

我的代码肯定有更多可以优化的地方，这里只是提供一个思路，希望对你有所帮助。

### 获取oss配置信息

```javascript
// 永久密钥
export const SecretId = 'AKIDSp145t****OBrJc2LSRFvOirlRs422tX';

// 存储桶
export const bucket = 'xxxx-cos-1329xxx581';

// 腾讯云COS地址
export const TX_COS_URL = `https://${bucket}.cos.ap-beijing.myqcloud.com/`;
```

## 直传设置上传文件路径

前端直传文件到腾讯OSS时，需要设置上传文件的路径，这里提供一个生成文件路径的方法。

我们的文件路径规则是：`模块/年/月/时间戳+uuid.文件后缀`，这样可以避免文件名重复。

```javascript
/**
 * 获取上传文件路径
 * @param {String} modulesType 文件模块
 * @param {String} filename 文件名
 * @returns {String} 生成的文件路径
 */
export const getFilePath = (modulesType, filename) => {
      return `${modulesType}/${new Date().getFullYear()}/${new Date().getMonth() + 1}/${new Date().getTime()}${uuidv4()}.${filename}`;
    };
```

### 上传文件

暴露一个上传文件的方法，传入文件模块、文件路径、文件名，返回一个Promise对象和一个上传路径拼接后的字符串。

```javascript

/**
 * 获取文件后缀
 * @param {String} fileName 文件名
 * @returns {String} 文件后缀
 */
function getFileExtension(fileName) {
  const parts = fileName.split('.');
  return parts.length > 1 ? parts.pop() : '';
}

/**
 * 上传文件
 * @param {String} modulesType 文件模块
 * @param {String} filePath 文件路径
 * @param {String} fileName 文件名
 * @returns {Promise<Object>} 上传结果
 */
export const uploadFile = (modulesType, filePath, fileName) => {
  return new Promise(async (resolve, reject) => {
    try {
      // 获取文件后缀
      const fileExtension = getFileExtension(fileName || filePath);
      // 获取根据规则生成的文件路径
      const key = getFilePath(modulesType, fileExtension);
      // 在这里设置上传路径，多文件上传时，需要将上传路径拼接成字符串，避免Promise.all重复提交
      const formData = {
        key,
        bucket,
        success_action_status: 200,
        'q-ak': SecretId,
        'q-key-time': new Date().getTime(),
      };

      // 上传文件
      const res = await uploadFileImage(filePath, formData)
      // 返回的上传路径拼接腾讯OSS地址
      res.Location = `${TX_COS_URL}${uploadPath}`;
      resolve(res);
    } catch (err) {
      reject(err);
    }
  });
};

/**
 * 上传文件
 * @param {String} filePath 文件路径
 * @param {Object} formData 上传表单数据
 * @returns {Promise<Object>} 上传结果
 */
const uploadFileImage = (filePath, formData) => {
  return new Promise((resolve, reject) => {
    // uni-app 上传文件
    uni.uploadFile({
      url: TX_COS_URL,
      filePath,
      name: 'file',
      formData,
      success: (res) => {
        if (![200, 204].includes(res.statusCode)) {
          reject(res);
        } else {
          resolve(res);
        }
      },
      fail: (err) => {
        reject(err);
      },
    });
  });
};

/**
 * 多文件上传
 * @param {Object} event 上传文件对象
 * @param {Array} uploadType 存储文件数组 传入fileList绑定的数组，直接返回使用不需要多余操作
 * @param {String} source 文件根路径 例：论坛'forum' 用户'user' 企业 'company' 等
 * @returns {Promise<Object>} 上传结果
 */
export const uploadFiles = (event, uploadType = [], source = 'user') => {
  return new Promise(async (resolve, reject) => {
    try {

      // 获取上传文件列表
      const lists = [].concat(event.file.res);

      // 上传文件放在Promise.all中，等待所有文件上传完成后，再返回结果
      const promises = lists.map((item) => {
        return uploadFile(source, item.url, item.name);
      });

      // promise.all返回的结果是一个数组，需要遍历处理
      const reusltUrl = await Promise.all(promises)

      // 处理上传结果
      reusltUrl.forEach((item) => {
        uploadType.push({
          ...item,
          status: 'success',
          message: '',
          url: item.Location,
        });
      });

      // 处理返回的上传路径
      const filePath = reusltUrl.map((item) => item.Location).join(',');

      // 返回上传结果 和 上传路径
      resolve({uploadData: uploadType, filePath});
    } catch (err) {
      reject(err);
    }
  });
};
```

## vue+element-ui 文件上传到腾讯OSS

vue+element-ui 上传文件到腾讯OSS，需要使用element-ui的上传组件，上传文件时需要设置上传路径、上传文件名、上传文件模块等信息。

```npx
    // 安装cos-js-sdk-v5
    npm install cos-js-sdk-v5 --save
```

### 获取oss配置信息

```javascript

// 文件名唯一标识拼接
import {v4 as uuidv4} from 'uuid';
// cos-js-sdk-v5 上传文件 
import COS from 'cos-js-sdk-v5';

// 永久密钥和key
const cos = new COS({
  SecretId: 'AKIDSp145*****c2LSRFvOirlRs422tX',
  SecretKey: '4ucM*********MtRsKixznMeiJl12oMg',
});
```

### 上传文件

```javascript

/**
 * 要暴露的方法
 * 获取上传文件路径
 * @param {String} modulesType 文件模块
 * @param {Object} file File对象
 */
export const getFilePath = (modulesType, file) => {
      return new Promise(async (resolve, reject) => {
        // oss文件路径拼接 上传到oss的文件路径
        const key = `${modulesType}/${new Date().getFullYear()}/${new Date().getMonth() + 1}/${new Date().getTime()}${uuidv4()}${file.name}`;
        const res = await uploadToCOS(file, key);
        resolve(res);
      })
    };

export function uploadToCOS(file, fileName) {
  return new Promise((resolve, reject) => {
    // 上传文件到腾讯OSS
    cos.putObject({
      Bucket: 'xxx-xx-132xxxx581', // 存储桶名称
      Region: 'ap-beijing',  // 存储桶所在地域
      Key: fileName,  // 文件路径
      Body: file, // 上传文件对象
      ContentLength: file.size, // 文件大小
    }, (err, data) => {
      if (err) {
        reject(err);
      } else {
        data.Location = 'https://' + data.Location; // 拼接https://上传路径
        resolve(data);
      }
    });
  });
}
```



