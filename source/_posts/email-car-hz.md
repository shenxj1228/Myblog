---
title: 发送杭州小客车摇号结果的邮件
date: 2018-04-04 16:40:11
tags: [Node,邮件,杭州小客车摇号]
---

*不知道为什么163邮箱老是说我内容非法，因此采用QQ邮箱*

## QQ邮箱开通SMTP
1. 打开QQ邮箱，进入账户选项

![image](http://blog.shenxj.xyz/markdown-image/qqmail-1.png)

![image](http://blog.shenxj.xyz/markdown-image/qqmail-2.png)
2. 开通SMTP服务

![image](http://blog.shenxj.xyz/markdown-image/qqmail-3.png)
3. 生成授权码

![image](http://blog.shenxj.xyz/markdown-image/qqmail-4.png)

## 查询接口解析

打开[杭州市小客车总量调控管理信息系统](http://xkctk.hangzhou.gov.cn/)，通过chrome捕获查询个人摇号结果请求的接口信息(不排除网站后期进行接口更新，需要重新获取)。
- 获取时间：2018-03-30
- 接口地址：http://apply.hzcb.gov.cn/apply/app/status/norm/person
- 接口参数：{pageNo: '1',issueNumber: '000000',applyCode: '123123123'}
- 接口报文头：'content-type': 'application/x-www-form-urlencoded'
- 接口方法：POST    

```
postOptions = {
        method: 'POST',
        uri: 'http://apply.hzcb.gov.cn/apply/app/status/norm/person',
        form: {
            pageNo: '1', //页码
            issueNumber: '000000', //月份，000000表示最近6个月
            applyCode: applyCode   //申请编码
        },
        headers: {
            'content-type': 'application/x-www-form-urlencoded'
        }
    }
```
#### 模拟请求

```
 rp({
    method: 'POST',
    uri: apiUrl,
    form: {
            pageNo: '1',
            issueNumber:'000000'
            applyCode: ''
        },
    headers: {
        'content-type': 'application/x-www-form-urlencoded'
    }
}).then(body => {
        //接口调用成功
        //Next处理
    })
  .catch(error=>{
        //接口调用失败
  })
```

---

## 定时发起

摇号结果一般在26日15点发布（周末顺延），因此需要判断是否工作日然后进行顺延，通过node-schedule模块进行计划任务，如果是周末，则新建一个计划任务周一进行查询，发起之后删除该计划

```
schedule.scheduleJob('* 05 15 26 * *', () => {
    let weekDay = new Date().getDay()
    if (weekDay === 0 || weekDay === 6) {
        let once = schedule.scheduleJob('* * * * * 1', () => {
            once.cancel()
           //查询摇号结果
        })
    } else {
        //查询摇号结果
    }
})
```

---

## 查询失败延时
先通过查询本月的日期是否已经发布，如果不存在进行延时半小时，再进行查询，直至查询次数满重试次数（默认4）次，才发邮件查询失败。

```
if (i < retryCount) {
    i++
    setTimeout(() => {
        //重新查询
    }, 30 * 60 * 1000)
} else {
    console.log(
        '\r\n超过最大重试次数，已停止，请手动查询：' + apiUrl
    )
}    
```

---

## 多邮箱
当一个申请编码对应多个邮箱时需要进行拼接处理，因此增加了一个配置文件，并重组对象
userinfo.js
```
const userinfos = [
    {
        mails: ['sxj123456789@qq.com', 'xl987654321@qq.com'],
        applyCode: '6666661888888'
    }
]
module.exports = userinfos

```


```
let tmpinfos = []
userinfos.forEach((item, index) => {
    let tos = item.mails.map(v => {
        return `"${v}" <${v}>`
    })
    let to = tos.join(',')
    let applyCode = item.applyCode
    tmpinfos.push({ to: to, applyCode: applyCode })
})
```

---

## 发送邮件

本次采用QQ邮箱STMP进行邮件发送。

```
const mailTransport = nodemailer.createTransport({
    host: 'smtp.qq.com',
    port: 465,
    secureConnection: true, // use SSL
    auth: {
        user: mailUser,
        pass: mailPass
    }
})
mailTransport.sendMail(mailoption, (err, msg) => {
    if (err) {
        console.log(err)
    }
})
```

---

## 完整代码
index.js

```
const cheerio = require('cheerio')
const rp = require('request-promise')
const schedule = require('node-schedule')
const nodemailer = require('nodemailer')
const userinfos = require('./userinfo.js')
const mailPass = 'xxxxxxx' //QQ邮箱授权码
const mailUser = '123456789@qq.com'
const retryCount = 4
const apiUrl = 'http://apply.hzcb.gov.cn/apply/app/status/norm/person'

const mailTransport = nodemailer.createTransport({
    host: 'smtp.qq.com',
    port: 465,
    secureConnection: true, // use SSL
    auth: {
        user: mailUser,
        pass: mailPass
    }
})
let j = schedule.scheduleJob('* 05 15 26 * *', () => {
    let weekDay = new Date().getDay()
    if (weekDay === 0 || weekDay === 6) {
        let once = schedule.scheduleJob('* * * * * 1', () => {
            once.cancel()
            check(1)
        })
    } else {
        check(1)
    }
})

console.log('杭州小客车摇号查询计划任务启动')

/**
 *入口
 *
 * @param {int} i 第N次查询
 */
function check(i) {
    console.log('\r\n' + new Date().toLocaleString() + ' - 日志：')
    getData()
        .then(infos => {
            infos.forEach((item, index) => {
                getCarIndex(item.applyCode, item.to)
            })
        })
        .catch(error => {
            if (i < retryCount) {
                i++
                setTimeout(() => {
                    check(i)
                }, 30 * 60 * 1000)
            } else {
                console.log(
                    '\r\n超过最大重试次数，已停止，请手动查询：' + apiUrl
                )
            }
        })
}
/**
 * 获取是否发布新的摇号结果
 *
 * @returns promise
 */
function getData() {
    return new Promise((resolve, reject) => {
        rp({
            method: 'POST',
            uri: apiUrl,
            form: {
                pageNo: '1',
                applyCode: ''
            },
            headers: {
                'content-type': 'application/x-www-form-urlencoded'
            }
        }).then(body => {
            const $ = cheerio.load(body, { decodeEntities: false })
            const newDate =
                new Date().getFullYear() +
                ('000' + (new Date().getMonth() + 1)).substr(-2)
            if (
                $('#issueNumber option[selected="selected"]').attr('value') ==
                newDate
            ) {
                let tmpinfos = []
                userinfos.forEach((item, index) => {
                    let tos = item.mails.map(v => {
                        return `"${v}" <${v}>`
                    })
                    let to = tos.join(',')
                    let applyCode = item.applyCode
                    tmpinfos.push({ to: to, applyCode: applyCode })
                })
                resolve(tmpinfos)
            } else {
                console.log('\r\n数据还未更新，未能找到日期：' + newDate)
                reject('data is not update')
            }
        })
    })
}

/**
 * 获取摇号结果
 *
 * @param {string} applyCode 申请编码
 * @param {string} to 发送邮箱
 */
function getCarIndex(applyCode, to) {
    console.log('\r\n查询申请编号：' + applyCode)
    rp({
        method: 'POST',
        uri: apiUrl,
        form: { pageNo: '1', issueNumber: '000000', applyCode: applyCode },
        headers: { 'content-type': 'application/x-www-form-urlencoded' }
    })
        .then(body => {
            const $ = cheerio.load(body, { decodeEntities: false })
            let mailOptions = {
                from: `"杭州小客车摇号结果" <${mailUser}>`,
                to: to,
                subject: '摇号结果：',
                html: ''
            } // bcc      : ''    //密送 // cc         : ''  //抄送
            if ($('tr.content_data>td').length < 2) {
                mailOptions.subject = mailOptions.subject + '未能中签'
            } else {
                mailOptions.subject = mailOptions.subject + '成功中签'
            }
            mailOptions.html =
                '<h2 style="color:red">查询结果：</h2> \r\n' +
                $.html('table.ge2_content')
            console.log(mailOptions)
            sendToMail(mailOptions)
        })
        .catch(err => {
            console.log('\r\n应答结果：' + err)
        })
}
/**
 * 发送邮件
 *
 * @param {string} mailoption 邮件内容选项
 */
function sendToMail(mailoption) {
    mailTransport.sendMail(mailoption, (err, msg) => {
        if (err) {
            console.log(err)
        }
    })
}

```


