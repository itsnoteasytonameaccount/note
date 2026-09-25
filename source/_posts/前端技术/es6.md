---
title: 用法笔记
tag: javascript
date: 2018/04/05 17:23:10
---

# 前言

为了熟练 promise 限制自己尽量用 promise 来解决自己平时的问题。
做个记录<!--more-->

# promise

## promise 异步深先

同学提的：怎么用 js 异步从任意点开始深先遍历一个矩阵

```javascript
//异步深度优先遍历(附加条件每步间隔1秒)
function start(x, y) {
  return new Promise((resolve, reject) => {
    setTimeout(function () {
      resolve();
      change(x, y);
    }, 1000);
  })
    .then(() => {
      if (
        area[x - 1] &&
        typeof area[x - 1][y] !== "undefined" &&
        !area[x - 1][y]
      ) {
        return new Promise((resolve, reject) => {
          start(x - 1, y).then(() => {
            resolve();
          });
        });
      }
    })
    .then(() => {
      if (typeof area[x][y - 1] !== "undefined" && !area[x][y - 1]) {
        return new Promise((resolve, reject) => {
          start(x, y - 1).then(() => {
            resolve();
          });
        });
      }
    })
    .then(() => {
      if (
        area[x + 1] &&
        typeof area[x + 1][y] !== "undefined" &&
        !area[x + 1][y]
      ) {
        return new Promise((resolve, reject) => {
          start(x + 1, y).then(() => {
            resolve();
          });
        });
      }
    })
    .then(() => {
      if (typeof area[x][y + 1] !== "undefined" && !area[x][y + 1]) {
        return new Promise((resolve, reject) => {
          start(x, y + 1).then(() => {
            resolve();
          });
        });
      }
    });
}
```

## sftp 上传文件夹

这个写得较早，用来把文件上传到服务器的，不熟练，代码较为难看

```javascript
//异步遍历文件夹所有子文件
"use strict";

const styles = require("./consoleStyle.js")();
const fs = require("fs");
const moment = require("moment");
class fileOp {
  constructor() {}
  getAllFileSync(dir) {
    let Files = [];
    fs.readdirSync(dir).forEach(function (ele) {
      let stats = fs.statSync(dir + "/" + ele);
      if (stats.isFile()) Files.push(dir + "/" + ele);
      else Files = Files.concat(this.getAllFileSync(dir + "/" + ele));
    }, this);
    return Files;
  }
  GetPromiseContentList(dir) {
    var Files = [];
    return new Promise(function (resolve, reject) {
      fs.readdir(dir, function (err, files) {
        if (err) reject(err);
        else {
          Files = Files.concat(files);
          resolve(Files);
        }
      });
    }).then(
      (data) => {
        return data;
      },
      (err) => {
        console.log(styles.red, err);
        return [];
      }
    );
  }
  getPromiseAllFileStats(dir) {
    var Files = [];
    var self = this;
    return new Promise((resolve, reject) => {
      fs.stat(dir, (err, stats) => {
        if (err) reject(err);
        else if (stats.isFile()) {
          let newObj = {};
          newObj.path = dir;
          newObj.updated = moment(
            stats.mtime,
            "ddd, DD MMM YYYY HH:mm:ss GMT"
          ).format("X");
          newObj.date = moment(
            stats.ctime,
            "ddd, DD MMM YYYY HH:mm:ss GMT"
          ).format("X");
          Files.push(newObj);
          resolve(Files);
        } else if (stats.isDirectory()) {
          self.GetPromiseContentList(dir).then((data) => {
            let i = 0;
            if (!data.length) return resolve([]);
            data.forEach((ele) => {
              self
                .getPromiseAllFileStats(dir + "/" + ele)
                .then((data2) => {
                  Files = Files.concat(data2);
                  i++;
                  return Promise.resolve(i);
                })
                .then(() => {
                  if (i == data.length) resolve(Files);
                });
            });
          });
        }
      });
    }).then(
      (data) => {
        return data;
      },
      (err) => {
        console.log(styles.red, err);
        return [];
      }
    );
  }
}
//异步sftp上传
function addDir(sftp, path, arr, file) {
  const HAS_CHILDREN = 0,
    DONT_HAS_CHILDREN = 1,
    MK_FAIL = 2;
  return new Promise((resolve, reject) => {
    //console.log(arr.length);
    if (arr.length) {
      sftp.list("/project/hexo/" + path).then(
        (res) => {
          resolve(HAS_CHILDREN);
        },
        (err) => {
          sftp
            .mkdir("/project/hexo/" + path, false)
            .then(() => {
              resolve(HAS_CHILDREN);
            })
            .catch((error) => {
              resolve(MK_FAIL);
            });
        }
      );
    } else {
      console.log("\x1B[32mINFO\x1B[39m  开始上传：" + file);
      sftp
        .put(file, "/project/hexo/" + path)
        .then(() => {
          resolve(DONT_HAS_CHILDREN);
        })
        .catch((err) => {
          resolve(MK_FAIL);
        });
    }
  }).then((res) => {
    if (res === HAS_CHILDREN) {
      return addDir(sftp, path + "/" + arr.pop(), arr, file).catch((err) => {
        console.log(err);
      });
    } else if (res === DONT_HAS_CHILDREN) {
      return;
    } else if (res === MK_FAIL) {
      console.log("创建文件夹或文件失败:", path);
    }
  });
}

let operation = new fileOp();

function sftpPost() {
  let sftp = new Client();
  sftp
    .connect({
      host: "",
      port: "",
      username: "",
      password: "",
    })
    .then(() => {
      new fileOp()
        .getPromiseAllFileStats(hexo.public_dir)
        .then((files) => {
          files.forEach((ele, index) => {
            let file = ele.path
              .replace(/(\/+)|(\\+)/g, "/")
              .replace(/\/+/g, "/");
            let arr = ele.path.split(hexo.public_dir)[1];
            arr = arr.split("/");
            arr = arr.reverse().slice(0, arr.length - 1);
            let path = arr.pop();
            addDir(sftp, path, arr, file).catch((err) => {
              console.error(err);
            });
          });
        })
        .catch((err) => {
          console.error(err);
        });
    })
    .then(() => {
      console.log("上传完成");
      sftp.end();
    })
    .catch((err) => {
      console.error(err);
      sftp.end();
    });
  return;
}
```

<!-- 小demo[深度优先遍历](http://www.flyingfishman.cn/demo/test.html). -->

# 阮一峰的尾递归优化代码注释

以下代码我觉得比较厉害，故记下来加上注释，省得下次看又不理解

```javascript
function tco(f) {
  var value;
  var active = false;
  var accumulated = []; //存储参数的栈

  return function accumulator() {
    accumulated.push(arguments); //压栈
    if (!active) {
      active = true; //访问栈的锁
      while (accumulated.length) {
        //栈内有参数则处理
        value = f.apply(this, accumulated.shift()); //弹栈作为参数
      }
      active = false; //释放锁
      return value;
    }
  };
}

var sum = tco(function (x, y) {
  if (y > 0) {
    return sum(x + 1, y - 1);
  } else {
    return x;
  }
});

sum(1, 100000);
//调用sum会 调用accumulator将参数保存，除第一次会进入循环外,sum执行的操作全都只是压栈。
//若此时active为true说明是在循环内部通过f调用的sum函数，故而不执行锁后的操作，仅保存参数供循环处理
//为false则说明没有在执行锁后的代码，说明是第一次调用，因而进入循环处理参数。
//以上代码等同于将递归调用化为循环调用
```
