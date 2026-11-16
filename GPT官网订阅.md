# 全流程操作步骤

## 1. 确认当前订阅彻底失效（回到免费版）

网页版点击cancle取消订阅

## 2. 检查并设置 Google 账号地区与付款资料

建议选择美区（美元结算，价格 20 美元）或日区（日元结算）：

### 2.1 查看与创建付款资料：

1. google play 谷歌商店绑定visa卡

2. 用手机或电脑浏览器打开 pay.google.com 并登录你的 Google 账号。

3. 点击右上角或菜单中的 “设置”（Settings），查看当前国家/地区。

4. 点击 “付款方式”（Payment methods） -> “添加付款方式”。

5. 账单地址填写要点：

   - 如果地区是美国，查看章节5： 账单地址建议填写美国免税州（如俄勒冈州 Oregon、阿拉斯加州 Alaska、蒙大拿州 Montana 或特拉华州 Delaware）的真实邮编和地址，能省去消费税。

   - 如果地区是日本，查看章节6： 账单地址填写一个标准的日本真实地址（如东京都某酒店地址）及对应的日本邮政编码。

6. 提交保存。Google 可能会发起一笔极小金额的预授权（随后会自动撤销）来验证卡片有效性。

## 3. 在手机 ChatGPT App 内完成订阅

### 3.1 网络环境准备：

1. 手机开启海外节点（确保节点地区支持 ChatGPT 服务，且建议与 Google Play 地区一致，如美区或日区）。
2. 准备 Google Play 商店：
   - 打开 Google Play 商店，点击右上角头像，确认当前登录的正是刚才绑定了 Visa 卡的 Google 账号。
   - 如果手机上有多个 Google 账号，最好先在系统设置中暂时移除其他账号，仅保留该账号，防止 App 内购时调用错误的账号通道。

### 3.1 发起订阅：

1. 打开 ChatGPT 手机 App。
2. 点击左上角菜单 -> 点击底部的个人设置 -> 选择 “Upgrade to Plus”。
3. 此时系统会弹起 Google Play 官方支付窗口，上面会明确标出扣款币种与金额（如 $20.00 或相应的日元金额）。
4. 点击 “订阅”（Subscribe），验证指纹或锁屏密码，即可顺利扣款开通。

## 4. 后续自动续费与管理

订阅成功后，后续每个月都会由 Google Play 自动从你的 Visa 卡中扣除当月费用，不再受 OpenAI 网页端 Stripe 风控的直接拦截。

后续如果想取消自动续订，直接进入 Google Play 商店 -> 点击右上角头像 -> 付款和订阅 -> 订阅，随时可以一键关闭。

## 5. 美区免税设置

免税修改步骤（直接在手机浏览器或电脑操作）进入 Google 支付中心设置：

1. 用浏览器访问并登录 pay.google.com。
2. 点击顶部或左侧菜单栏的 “设置”（Settings）
3. 修改法定地址（Legal address）：
   - 在“付款资料”（Payments profile）下方找到 “地址”（Address） 或 “法定地址”（Legal address），点击旁边的铅笔（编辑）图标。
   - 将地址改为美国免税州（最常用的是俄勒冈州 Oregon，全州 0% 消费税）：
     - 州（State）： Oregon（或缩写 OR）
     - 邮编（ZIP code）： 97201（或 97204、97035）
     - 城市（City）： Portland
     - 街道地址（Street address）： 可以填当地任意真实街区或酒店地址（例如 1000 SW Broadway）。
     - 点击 “保存”（Save）。
4. 修改绑定的 Visa 卡账单地址：
   - 点击顶部菜单的 “付款方式”（Payment methods）。找到截图里显示的尾号 1313 这张 Visa 卡，点击 “修改”（Edit）。   
   - 将该卡关联的账单地址同样更新为上述俄勒冈州（Oregon）的地址与邮编 97201，点击保存。
5. 重新拉起支付验证
   - 返回手机桌面，彻底关闭并滑掉 ChatGPT App 和 Google Play 商店 的后台运行。
   - 重新打开 ChatGPT App，进入订阅升级界面，再次点击 Upgrade for $19.99。   
   - 此时弹出的 Google Play 确认窗口中，$1.32 的税费（includes tax）会直接归零消失，总计扣款金额会变为干净的 $19.99/月，直接点击 Subscribe 确认即可。

## 6. 日本订阅设置

### 6.1 取消当前订阅的自动续费（不会影响本月正常使用）

必须先关闭自动续订，否则系统到期会直接继续扣除美元：

1. 打开 Google Play Store App。
2. 点击右上角 profile icon（头像图标）。
3. 进入 Payments & subscriptions $\rightarrow$ Subscriptions。
4. 找到并点击 ChatGPT。
5. 滑动到底部点击 Cancel subscription。
6. 根据屏幕提示选择取消原因，点击 Continue $\rightarrow$ 确认 Cancel subscription。

注意： 取消后，你当前的 Plus 权益依然可以正常使用到本月最后一天（到期日可以在该页面查看）。

### 6.2: 在 Google 支付中心建立 Japan 付款资料并绑卡

Google 规定一个付款资料（Payments profile）只能对应一个固定国家，需要添加日本付款资料：

1. 使用手机浏览器或电脑打开并登录 pay.google.com。
2. 点击菜单栏 Settings 选项卡。
3. 找到 Payments profile，点击旁边的 Edit（铅笔图标）。
4. 在 Country/Region 下拉菜单中选择 Japan (JP)。
5. 输入日本账单地址（Name and address）：  
   - Name： 输入你的名字拼音（如卡片上的英文名字）。
   - Address line 1： Ushizucho Ushizu（牛津町牛津）。
   - Address line 2: 可以留空，或填入具体门牌号/建筑名（如 1-1）
   - City： Ogi-shi（或直接填 Ogi）。
   - Prefecture： Saga（佐賀県，下拉菜单选择）
   - Postal code： 输入 7 位日本邮编849-0303
6. 点击 Submit 完成创建。
7. 切换到菜单栏的 Payment methods 选项卡：（同美区修改绑定VISA卡账单地址）
   - 确保当前正在查看的是刚建好的 Japan 付款资料。
   - 点击 Add payment method。
   - 输入你的国内 Visa 双币卡卡号、有效期（Exp date）与 CVC 安全码，保存完成绑定。

### 6.3 等待到期，重新在手机 App 发起日元订阅

1. 等待本月周期结束：
   - 等到下个月当前周期结束、ChatGPT 恢复为 Free 免费版（页面重新出现 Upgrade 按钮）。
2. 手机网络环境准备：手机开启日本（Japan）节点。
3. 刷新 Google Play 商店地区：
   - 打开 Google Play Store，点击右上角 profile icon $\rightarrow$ Settings $\rightarrow$ General $\rightarrow$ Account and device preferences。
   - 在 Country and profiles 下确认当前已切换为 Japan。
4. 重新订阅：
   - 打开 ChatGPT App，进入设置选择 Upgrade to Plus。   
   - 此时弹出的 Google Play 购买弹窗，标价会变成日元形式（如 ¥3,000 或含税 ¥2,727 左右），不再显示美元符号。
   - 选择你在 Japan profile 下绑定的 Visa 卡，点击 Subscribe 完成指纹/密码验证。
