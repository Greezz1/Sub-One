<script setup>
import { ref, computed, watch, onMounted } from 'vue';

const props = defineProps({
    modelValue: {
        type: String,
        default: ''
    }
});

const emit = defineEmits(['update:modelValue']);

// 预定义数据
// 预定义数据
const protocols = [
    { label: 'Shadowsocks', value: 'ss' },
    { label: 'VMess', value: 'vmess' },
    { label: 'VLESS', value: 'vless' },
    { label: 'Trojan', value: 'trojan' },
    { label: 'Hysteria2', value: 'hysteria2' },
    { label: 'Tuic', value: 'tuic' },
    { label: 'Socks5', value: 'socks5' },
    { label: 'WireGuard', value: 'wg|wireguard' },
    { label: 'Reality', value: 'reality' }
];

const regions = [
    { label: '🇭🇰 香港', value: 'HK|Hong Kong|HongKong|香港|Hong K' },
    { label: '🇹🇼 台湾', value: 'TW|Taiwan|Tai Wan|台湾|臺灣|台北|Taipei' },
    { label: '🇸🇬 新加坡', value: 'SG|Singapore|Singpore|新加坡|狮城|SGP' },
    { label: '🇯🇵 日本', value: 'JP|Japan|日本|东京|Tokyo|Osaka|大阪' },
    { label: '🇺🇸 美国', value: 'US|United States|America|USA|美国|美國|洛杉矶|Los Angeles|San Jose|New York' },
    { label: '🇰🇷 韩国', value: 'KR|Korea|South Korea|韩国|韓國|首尔|Seoul' },
    { label: '🇨🇳 中国', value: 'CN|China|中国|回国|内地|江苏|北京|上海|广州|深圳|杭州' },
    { label: '🇬🇧 英国', value: 'GB|UK|United Kingdom|Britain|英国|伦敦|London' },
    { label: '🇩🇪 德国', value: 'DE|Germany|Deutschland|德国|法兰克福|Frankfurt' },
    { label: '🇦🇺 澳洲', value: 'AU|Australia|澳洲|澳大利亚|悉尼|Sydney' },
    { label: '🇨🇦 加拿大', value: 'CA|Canada|加拿大|多伦多|Toronto|Vancouver' },
    { label: '🇮🇳 印度', value: 'IN|India|印度|孟买|Mumbai' },
    { label: '🇷🇺 俄罗斯', value: 'RU|Russia|俄罗斯|莫斯科|Moscow' },
    { label: '🇫🇷 法国', value: 'FR|France|法国|巴黎|Paris' },
    { label: '🇳🇱 荷兰', value: 'NL|Netherlands|Holland|荷兰|阿姆斯特丹' }
];

const commonKeywords = [
    '高倍率', '低倍率', '中转', '直连', '专线', 'BGP', 'IPLC', 'IEPL',
    'IPv6', 'UDP', '游戏', '流媒体', '解锁',
    '过期', '官网', '剩余流量', '到期', '重置', '测试'
];

// 状态
const mode = ref('exclude'); // 'exclude' | 'keep'
const selectedProtocols = ref([]);
const selectedRegions = ref([]);
const customKeywords = ref([]);
const newKeyword = ref('');
const isManualMode = ref(false);

// 解析逻辑：尝试从字符串还原状态
const parseValue = (val) => {
    if (!val) return;

    // 简单启发式检查：如果包含复杂的正则符号（除了我们生成的），则切换到手动模式
    // 这里暂时只做简单解析，解析失败保留在手动模式

    const lines = val.split('\n').map(l => l.trim()).filter(l => l);
    if (lines.length === 0) return;

    // 检测模式
    const hasKeep = lines.some(l => l.startsWith('keep:'));
    mode.value = hasKeep ? 'keep' : 'exclude';

    const cleanLines = lines.map(l => l.replace(/^keep:/, ''));

    let foundProtocols = [];
    let foundRegions = [];
    let foundKeywords = [];

    cleanLines.forEach(line => {
        if (line.startsWith('proto:')) {
            const protos = line.replace('proto:', '').split(',');
            foundProtocols.push(...protos);
        } else {
            // 尝试匹配地区
            let matchedRegion = false;
            for (const r of regions) {
                if (line === `(${r.value})` || line === r.value) {
                    foundRegions.push(r.value);
                    matchedRegion = true;
                    break;
                }
            }

            if (!matchedRegion) {
                // 去除正则括号
                const cleanKey = line.replace(/^\(/, '').replace(/\)$/, '');
                // 分割可能的多选
                const keys = cleanKey.split('|');
                foundKeywords.push(...keys);
            }
        }
    });

    selectedProtocols.value = [...new Set(foundProtocols)];
    selectedRegions.value = [...new Set(foundRegions)];
    customKeywords.value = [...new Set(foundKeywords)];
};

// 生成逻辑：状态 -> 字符串
const generateString = () => {
    if (isManualMode.value) return props.modelValue;

    const lines = [];
    const prefix = mode.value === 'keep' ? 'keep:' : '';

    // 1. 协议
    if (selectedProtocols.value.length > 0) {
        lines.push(`${prefix}proto:${selectedProtocols.value.join(',')}`);
    }

    // 2. 地区 (每个地区单独一行，或者合并？为了清晰，合并成一个正则)
    // 为了匹配准确，使用 (A|B|C) 格式
    if (selectedRegions.value.length > 0) {
        const regionPattern = selectedRegions.value.join('|');
        lines.push(`${prefix}(${regionPattern})`);
    }

    // 3. 自定义关键词
    if (customKeywords.value.length > 0) {
        const keywordPattern = customKeywords.value.join('|');
        lines.push(`${prefix}(${keywordPattern})`);
    }

    return lines.join('\n');
};

// 监听状态变化，更新 modelValue
watch([mode, selectedProtocols, selectedRegions, customKeywords], () => {
    if (!isManualMode.value) {
        emit('update:modelValue', generateString());
    }
}, { deep: true });

// 初始化
onMounted(() => {
    if (props.modelValue) {
        // 尝试解析，如果看起来像手写的复杂正则，则保持手动模式
        // 这里简单处理：总是尝试解析，解析不出来的部分会变成 customKeywords
        parseValue(props.modelValue);
    }
});

// 添加自定义关键词
const addKeyword = () => {
    if (newKeyword.value.trim()) {
        customKeywords.value.push(newKeyword.value.trim());
        newKeyword.value = '';
    }
};

const removeKeyword = (k) => {
    customKeywords.value = customKeywords.value.filter(item => item !== k);
};

const toggleRegion = (rValue) => {
    const index = selectedRegions.value.indexOf(rValue);
    if (index === -1) {
        selectedRegions.value.push(rValue);
    } else {
        selectedRegions.value.splice(index, 1);
    }
};

const toggleProtocol = (pValue) => {
    const index = selectedProtocols.value.indexOf(pValue);
    if (index === -1) {
        selectedProtocols.value.push(pValue);
    } else {
        selectedProtocols.value.splice(index, 1);
    }
};

const clearAll = () => {
    selectedProtocols.value = [];
    selectedRegions.value = [];
    customKeywords.value = [];
};
</script>

<template>
    <div class="bg-gray-50 dark:bg-gray-900/50 rounded-xl border border-gray-200 dark:border-gray-700 p-4 space-y-4">

        <!-- 模式切换 -->
        <div class="flex items-center justify-between">
            <div class="flex bg-gray-200 dark:bg-gray-800 rounded-lg p-1">
                <button @click="mode = 'exclude'"
                    class="px-4 py-1.5 rounded-md text-sm font-medium transition-all duration-200"
                    :class="mode === 'exclude' ? 'bg-white dark:bg-gray-600 text-red-600 shadow-sm' : 'text-gray-500 hover:text-gray-700 dark:text-gray-400'">
                    🚫 排除模式 (黑名单)
                </button>
                <button @click="mode = 'keep'"
                    class="px-4 py-1.5 rounded-md text-sm font-medium transition-all duration-200"
                    :class="mode === 'keep' ? 'bg-white dark:bg-gray-600 text-green-600 shadow-sm' : 'text-gray-500 hover:text-gray-700 dark:text-gray-400'">
                    ✅ 保留模式 (白名单)
                </button>
            </div>

            <button @click="clearAll" class="text-xs text-gray-400 hover:text-red-500 underline">
                清空所有规则
            </button>
        </div>

        <!-- 协议选择 -->
        <div>
            <label class="text-xs font-semibold text-gray-500 uppercase tracking-wider mb-2 block">协议类型</label>
            <div class="flex flex-wrap gap-2">
                <button v-for="p in protocols" :key="p.value" @click="toggleProtocol(p.value)"
                    class="px-3 py-1.5 rounded-lg text-xs font-medium border transition-all duration-200"
                    :class="selectedProtocols.includes(p.value)
                        ? 'bg-indigo-50 dark:bg-indigo-900/30 border-indigo-200 dark:border-indigo-700 text-indigo-600 dark:text-indigo-400'
                        : 'bg-white dark:bg-gray-800 border-gray-200 dark:border-gray-700 text-gray-600 dark:text-gray-400 hover:border-gray-300'">
                    {{ p.label }}
                </button>
            </div>
        </div>

        <!-- 地区选择 -->
        <div>
            <label class="text-xs font-semibold text-gray-500 uppercase tracking-wider mb-2 block">常用地区</label>
            <div class="flex flex-wrap gap-2">
                <button v-for="r in regions" :key="r.value" @click="toggleRegion(r.value)"
                    class="px-3 py-1.5 rounded-lg text-xs font-medium border transition-all duration-200"
                    :class="selectedRegions.includes(r.value)
                        ? 'bg-emerald-50 dark:bg-emerald-900/30 border-emerald-200 dark:border-emerald-700 text-emerald-600 dark:text-emerald-400'
                        : 'bg-white dark:bg-gray-800 border-gray-200 dark:border-gray-700 text-gray-600 dark:text-gray-400 hover:border-gray-300'">
                    {{ r.label }}
                </button>
            </div>
        </div>

        <!-- 关键词 -->
        <div>
            <label class="text-xs font-semibold text-gray-500 uppercase tracking-wider mb-2 block">关键词 (名称包含)</label>

            <!-- 常用词 -->
            <div class="flex flex-wrap gap-2 mb-3">
                <button v-for="k in commonKeywords" :key="k"
                    @click="!customKeywords.includes(k) ? customKeywords.push(k) : removeKeyword(k)"
                    class="px-2 py-1 rounded text-xs border border-dashed transition-all duration-200" :class="customKeywords.includes(k)
                        ? 'bg-amber-50 border-amber-300 text-amber-600'
                        : 'border-gray-300 text-gray-500 hover:border-gray-400'">
                    {{ k }}
                </button>
            </div>

            <!-- 自定义输入 -->
            <div class="flex gap-2">
                <input v-model="newKeyword" @keyup.enter="addKeyword" type="text" placeholder="输入关键词后回车..."
                    class="flex-1 px-3 py-2 bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-indigo-500">
                <button @click="addKeyword"
                    class="px-4 py-2 bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-300 rounded-lg text-sm hover:bg-gray-200 dark:hover:bg-gray-600">
                    添加
                </button>
            </div>

            <!-- 已选关键词 -->
            <div v-if="customKeywords.length > 0" class="flex flex-wrap gap-2 mt-3">
                <span v-for="k in customKeywords" :key="k"
                    class="inline-flex items-center px-2.5 py-1 rounded-md text-xs font-medium bg-gray-100 dark:bg-gray-700 text-gray-700 dark:text-gray-300 border border-gray-200 dark:border-gray-600">
                    {{ k }}
                    <button @click="removeKeyword(k)" class="ml-1.5 text-gray-400 hover:text-red-500">×</button>
                </span>
            </div>
        </div>

        <!-- 预览/手动编辑 -->
        <div class="pt-2 border-t border-gray-200 dark:border-gray-700">
            <div class="flex items-center justify-between mb-2">
                <label class="text-xs font-semibold text-gray-500">生成的规则预览</label>
                <button @click="isManualMode = !isManualMode" class="text-xs text-indigo-500 hover:underline">
                    {{ isManualMode ? '切换回可视化模式' : '切换到手动编辑模式' }}
                </button>
            </div>
            <textarea :value="modelValue" @input="$emit('update:modelValue', $event.target.value)"
                :readonly="!isManualMode" rows="3"
                class="w-full px-3 py-2 bg-gray-100 dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-lg text-xs font-mono text-gray-600 dark:text-gray-400 focus:outline-none"
                :class="{ 'opacity-75 cursor-not-allowed': !isManualMode }"></textarea>
        </div>

    </div>
</template>
