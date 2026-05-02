<template>
  <scroll-view class="legal-page" scroll-y>
    <view class="content">
      <text class="title">隐私政策</text>
      <text class="meta">更新日期：2026-05-01</text>

      <text class="section-title">1. 我们收集的数据</text>
      <text class="paragraph">当你注册、登录和使用 AI 对话、图片识别、绘图等能力时，应用会处理账号信息、聊天内容、上传图片、设备基础网络信息以及算力记录。</text>

      <text class="section-title">2. 数据用途</text>
      <text class="paragraph">这些数据仅用于账号登录、保存会话历史、扣费记录、问题排查、滥用防控，以及向第三方模型服务商发起你主动触发的 AI 请求。</text>

      <text class="section-title">3. 第三方处理</text>
      <text class="paragraph">当你使用 AI 能力时，输入内容和上传图片可能会发送给第三方模型服务商处理。目前生产能力可能涉及火山引擎豆包相关模型服务，以及 Google Gemini 相关模型服务。请不要上传身份证、银行卡、医疗报告等高敏感数据。</text>

      <text class="section-title">4. 数据存储与删除</text>
      <text class="paragraph">会话记录、余额日志和举报记录可能在服务端保存，直到你主动删除账号或运营方基于风控、合规、排障需要进行清理。</text>

      <text class="section-title">5. 你的权利</text>
      <text class="paragraph">你可以在“我的”页面查看账号信息、退出登录，并通过“删除账号”功能删除账号及其相关数据。</text>

      <text class="section-title">6. AI 输出说明</text>
      <text class="paragraph">AI 生成内容可能不准确、不完整或不适用于专业决策。医疗、法律、金融等场景请勿直接依赖 AI 输出。</text>

      <text class="section-title">7. 联系方式</text>
      <text class="paragraph">如果你对隐私政策、数据处理或账号删除有疑问，可以通过以下邮箱联系开发者：819317636@qq.com</text>

      <text class="section-title">7. 上线前需补齐项</text>
      <text class="paragraph">当前页面用于提供应用内隐私入口。正式上架前，请将同版本隐私政策部署为公开 HTTPS 页面，并在 Google Play Console 中填写一致的隐私政策链接。</text>
    </view>
  </scroll-view>
</template>

<script setup>
</script>

<style scoped>
.legal-page { min-height: 100vh; background-color: var(--bg-page); }
.content { padding: 20px 16px 40px; }
.title { display: block; font-size: 24px; font-weight: 700; color: var(--text-primary); margin-bottom: 8px; }
.meta { display: block; font-size: 12px; color: var(--text-tertiary); margin-bottom: 20px; }
.section-title { display: block; font-size: 16px; font-weight: 600; color: var(--text-primary); margin: 18px 0 8px; }
.paragraph { display: block; font-size: 14px; line-height: 1.7; color: var(--text-secondary); }
</style>
