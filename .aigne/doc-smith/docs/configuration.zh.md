# 配置

Tokio 旨在实现灵活性和高效率，允许你根据具体应用需求进行定制。正确的配置能够显著影响项目的编译时间、二进制文件大小和可用功能。本节将引导你了解配置 Tokio 的关键方面，从选择功能标志、理解平台特定行为到选择性启用实验性 API。

<x-cards>
  <x-card data-title="功能标志" data-icon="lucide:toggle-right" data-href="/configuration/features">
    了解如何使用功能标志以仅包含必要的组件，从而优化应用的性能和依赖。
  </x-card>
  <x-card data-title="平台支持" data-icon="lucide:laptop" data-href="/configuration/platforms">
    查看官方支持的平台列表，并了解针对 WASM 等环境的特定注意事项。
  </x-card>
  <x-card data-title="不稳定功能" data-icon="lucide:flask-conical" data-href="/configuration/unstable">
    探索如何启用和使用实验性功能以获取前沿功能，同时注意潜在的 API 变更。
  </x-card>
</x-cards>