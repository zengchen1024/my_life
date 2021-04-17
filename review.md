---


---

<h1 id="approve，lgtm的重构">Approve，LGTM的重构</h1>
<p>标签（空格分隔）： yabot</p>
<hr>
<h2 id="背景">背景</h2>
<p>代码review的环境已经从专业的review网站转到代码托管平台上来，希望能延续专业review网站的用户体验。</p>
<h2 id="方案">方案</h2>
<p>社区的成员角色分为3类， 仓库member， reviewer， approver，见<a href="https://github.com/kubernetes/community/blob/master/community-membership.md">角色</a>。</p>

<table>
<thead>
<tr>
<th align="left">指令</th>
<th align="left">描述</th>
<th>谁能使用</th>
</tr>
</thead>
<tbody>
<tr>
<td align="left">/lgtm — looks good to me</td>
<td align="left">认可代码</td>
<td>任何人，非reviewer的评论不影响最终的结果</td>
</tr>
<tr>
<td align="left">/lbtm — looks bad to me</td>
<td align="left">不认可代码，希望作者修改</td>
<td>任何人，非reviewer的评论不影响最终的结果</td>
</tr>
<tr>
<td align="left">/approve</td>
<td align="left">认可代码</td>
<td>approver</td>
</tr>
<tr>
<td align="left">/reject</td>
<td align="left">不认可代码</td>
<td>approver</td>
</tr>
</tbody>
</table><p>approver和reviewer的工作都是review代码，只不过approver有合入权限。</p>
<p>一个pr能够合入，每个模块至少有2个不同的人approve且无reject。有reject，需要pr的作者与该approver沟通，让其取消reject。</p>
<p>lgtm标签只作为approver review代码的参考，不是pr合入的必要条件。</p>
<p>approver 评论/approve不能代替/lgtm，/reject也不能代替/lbtm。一旦approve标签打上，再评论/lgtm和/lbtm，只记录评论人但不影响lgtm标签。</p>
<p>一旦approve和ci-test-success标签打上，启动pr的合入流程。</p>
<h2 id="pr的合入过程">PR的合入过程</h2>
<h3 id="pr合入的状态图">PR合入的状态图</h3>
<p><img src="https://github.com/zengchen1024/test-infra/blob/master/prow/docs/review_process.png" alt="enter image description here"></p>
<h3 id="pr合入过程详解">PR合入过程详解</h3>
<p>一个pr合入，应该按如下的顺序打上对应的标签<br>
can_review    -------------&gt;   lgtm    -------------&gt;   ci-test-success    -------------&gt;   approve</p>
<ol>
<li>pr提交后自动跑A类用例；通过后打上 <strong>can_review</strong> 的标签，否则打上 <strong>ci-test-failure</strong> 标签。pr作者如果看<strong>ci-test-failure</strong> 标签，应该修改pr，或retest（pr作者对用例跑失败后的操作是一致的）</li>
</ol>
<pre><code>   用例分成3类：A) 代码格式相关的，B) 其他测试用例(比较重型的用例,功能性测试用例等)
   
   pr代码有修改应该自动跑A类用例；当master分支有变化时，如果有/lgtm标签，应该跑B类用例
</code></pre>
<ol start="2">
<li>reviewer看到 <strong>can_review</strong> 的标签则可开始review。</li>
</ol>
<pre><code>   打上lgtm标签时，删除 can_review 标签并自动评论/ok-to-test 标签， 启动跑B类用例。

   任何人在pr open状态下的任何时间点都可以来review，机器人记录下reviewer。
   approve标签打上后，reviewer评论任意 /lgtm， /lbtm，将不会生效，只会记录reviewer。
      
   在ci-test-success 标签打上后，approve标签才能打上；can-review标签打上之后，lgtm标签才能打上。
   如果没有配置相应的测试用例，则只要满足review的条件，lgtm和approve标签就可以打上。
   
   凡是修改了代码，则ci-test-success，approve，lgtm和can_review标签都要删除；重新跑B类测试用例时，ci-test-*相关的标签和approve标签都应该删除
   
   如果自动评论/ok-to-test失败，只要是仓库的member都可手动评论/ok-to-test，触发执行用例

   pr修改清空以前所有的review记录不合适，因为pr作者可能只修改了其中某个模块的代码，清空以前的所有记录就需要重新review；
   理想的情况是只对修改涉及到的模块重新review，不涉及的就不用重新review。那么有没有方式知道用户这次提交了哪些commit呢？   
</code></pre>
<ol start="3">
<li>
<p>所有测试通过之后，打上 <strong>ci-test-success</strong> 标签；否则打上 <strong>ci-test-failure</strong> 标签，重复步骤1， 2， 3</p>
</li>
<li>
<p>approver对打上 <strong>lgtm</strong> 和 <strong>ci-test-success</strong> 标签的pr进行review</p>
</li>
</ol>
<h2 id="判断是否满足lgtm-标签的方法">判断是否满足lgtm 标签的方法</h2>
<p>整个代码仓按目录结构配置OWNERS文件，文件中配置approver和reviewer，一个目录树中approver、reviewer的权限范围大小是，父目录的approver，reviewer &gt; 子目录。</p>
<p>一个PR中每个修改目录的得分均 &gt;= 1时，lgtm标签将被打上。</p>
<p>目录的review得分 = /lgtm 的总数（reviewer） + /approve 的总数 - /lbtm的总数（reviewer）；如果有/reject涉及此目录，则分数永远为负数</p>
<p>/lbtm的评论由approver来决定是否采纳，如果采纳，其应该评论/reject。</p>
<h2 id="判断是否满足approve-标签的方法">判断是否满足approve 标签的方法</h2>
<p>一个PR中每个修改目录的得分均 &gt;= 1时，approve标签将被打上。</p>
<p>目录的approve得分 = /approve 的总数；如果有/reject涉及此目录，则分数永远为负数</p>
<h3 id="例子">例子</h3>
<p>代码中的OWNERS文件如下</p>
<pre><code># ls -l
OWNERS
a/
a/OWNERS
b/
b/OWNERS

# cat ./OWNERS
reviwers:
- zhangsan

# cat a/OWNERS
reviewers:
- lisi
- xwz

# cat b/OWNERS
reviewers:
- wangwu
- cz
- ocl
</code></pre>
<p>假设一个pr修改了a， b两个目录下的文件，下面2个例子来说明lgtm标签是怎么打上的。</p>
<p>2.1 场景一</p>
<ul>
<li>zhangsan 评论了/lgtm，因为zhangsan是根目录的reviewer，他的review范围包含整个代码，因此lgtm直接可以加上</li>
</ul>
<pre><code>   /lgtm: zhangsan
   
   The lgtm label is added.
   
   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS  
</code></pre>
<p>2.1 场景二</p>
<ul>
<li>cuihua 评论了/lgtm，因为cuihua不是reviewer，因此不会加上lgtm的标签， 机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua
   
   Needs reviewers to review the codes in each of following directories before adding lgtm label:
   /a/OWNERS
   /b/OWNERS  

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS  
</code></pre>
<ul>
<li>现在lisi 加了/lgtm 的评论， 因为lisi是/a目录（模块）的reviewer，但/b目录下的reviewer还没有加/lgtm的评论，因此不会加上lgtm的标签，机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua, lisi
   
   Needs reviewers to review the codes in each of following directories before adding lgtm label:
   /b/OWNERS  

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS 
</code></pre>
<ul>
<li>现在wangwu 加了/lgtm 的评论， 因为wangwu 是/b目录（模块）的reviewer，且现在只剩下/b目录下的代码需要review，因此加上lgtm 的标签，机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua, lisi, wangwu
   
   The lgtm label is added.

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS 
</code></pre>
<p>2.2 场景三</p>
<ul>
<li>cuihua 评论了/lgtm，因为cuihua不是reviewer，因此不会加上lgtm的标签， 机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua
   
   Needs reviewers to review the codes in each of following directories before adding lgtm label:
   /a/OWNERS
   /b/OWNERS  

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS  
</code></pre>
<ul>
<li>现在lisi 加了/lgtm 的评论， 因为lisi是/a目录（模块）的reviewer，但/b目录下的reviewer还没有加/lgtm的评论，因此不会加上lgtm的标签，机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua, lisi
   
   Needs reviewers to review the codes in each of following directories before adding lgtm label:
   /b/OWNERS  

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS  
</code></pre>
<ul>
<li>现在wangwu 加了/lbtm 的评论，因为wangwu 是/b目录（模块）的reviewer，机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua, lisi
   /lbtm: wamgwu

   It can't add lgtm label.

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS  
</code></pre>
<ul>
<li>现在wangwu 加了/lgtm 的评论， 因为wangwu 是/b目录（模块）的reviewer，因此加上lgtm 的标签，机器人回复如下格式的评论</li>
</ul>
<pre class=" language-markdown"><code class="prism  language-markdown">   /lgtm: cuihua, lisi, wangwu
   
   The lgtm label is added.

   Needs approvers to approve the codes in each of following directories before adding approved label:
   /a/OWNERS
   /b/OWNERS 
</code></pre>

