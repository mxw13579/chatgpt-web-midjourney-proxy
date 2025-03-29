<script setup lang="ts">
import { ref ,computed,watch,onMounted} from 'vue';
import {useMessage, NButton,NSelect,NInput,NTag,NPopover,NInputNumber} from 'naive-ui';
import {gptFetch, mlog, upImg} from '@/api';
import { homeStore } from '@/store';
import { SvgIcon } from '@/components/common';
import config from "./draw.json";
import {t} from "@/locales";

const ms = useMessage();
const modelConfig = ref( {
model:[
{  "label": "DALL·E 3", "value": "dall-e-3" }
 ,{  "label": "DALL·E 2", "value": "dall-e-2" }
 ,{  "label": "Flux", "value": "flux" }
 ,{  "label": "Flux-Dev", "value": "flux-dev" }
 ,{  "label": "Flux-Pro", "value": "flux-pro" }
 ,{  "label": "midjourney-v6.1", "value": "midjourney-v6.1" }
 ,{  "label": "midjourney-v6", "value": "midjourney-v6" }
 ,{  "label": "midjourney-v5.2", "value": "midjourney-v5.2" }
 ,{  "label": "Flux.1.1-Pro", "value": "flux.1.1-pro" }
]
});
const st =ref({isGo:false });
const f = ref({size:'1024x1024', prompt:'',"model": "dall-e-3","n": 1, bili:-1, quality:'',view:'',light:'',shot:'',style:'', styles:'',version:'',sref:'',cref:'',cw:''});

const vf=[{s:'width: 100%; height: 100%;',label:'1:1'}
,{s:'width: 100%; height: 75%;',label:'4:3'}
,{s:'width: 75%; height: 100%;',label:'3:4'}
,{s:'width: 100%; height: 50%;',label:'16:9'}
,{s:'width: 50%; height: 100%;',label:'9:16'}
];

const farr= [
{ k:'style',v:t('mjchat.tStyle') }
,{ k:'view',v: t('mjchat.tView') }
,{ k:'shot',v: t('mjchat.tShot') }
,{ k:'light',v: t('mjchat.tLight') }
,{ k:'quality',v: t('mjchat.tQuality') }
,{ k:'styles',v:t('mjchat.tStyles') }
// ,{ k:'version',v:t('mjchat.tVersion') }
];

const drawlocalized = computed(() => {
	let localizedConfig = {};
	Object.keys(config).forEach((key) => {
		localizedConfig[key] = config[key].map((option) => {
			// 假设 labelKey 如 "draw.qualityList.general"
			let path = option.labelKey; // 直接使用 labelKey 作为路径
			return {
				...option,
				label: t(path), // 从 i18n 中获取本地化的标签
			};
		});
	});
	return localizedConfig;
});
const isDisabled= computed(()=>{
    if(st.value.isGo) return true;
    if(f.value.prompt.trim()=='') return true;
    return false;
});
const create= async ()=>{
    // 判断是否为midjourney模型
    if (f.value.model.startsWith('mid')) {
        // 处理midjourney模型的提示词
        let prompt = f.value.prompt.trim();
        let drawText = createPrompt(prompt);
        const rz = { prompt: prompt, drawText: drawText };


			//通过 f.value 生成新的变量
			let newF = { ...f.value };
			newF.prompt = rz.drawText;
			newF.quality = '';

        let obj = {
            action: 'gpt.dall-e-3',
            data: newF
        };
			homeStore.setMyData({act:'draw', actData:obj});
    } else {
			console.log(f)
        // 处理其他模型
        let obj = {
            action: 'gpt.dall-e-3',
            data: f.value
        };

        homeStore.setMyData({act:'draw', actData:obj});

    }

    st.value.isGo=true;
}
watch(()=>homeStore.myData.act,(n)=>{
	console.log(watch)
	console.log(n)
    if(n=='dallReload') {
        st.value.isGo=false;
        f.value.prompt='';
    }
    if(n=='updateChat')  st.value.isGo=false;
})

const dimensionsList= computed(()=>{
    if(f.value.model=='dall-e-2'){
        return [{
                "label": "1024px*1024px",
                "value": "1024x1024"
            }, {
                "label": "512px*512px",
                "value": "512x512"
            }, {
                "label": "256px*256px",
                "value": "256x256"
            }
    ];
    }
    return [{
                "label": "1024px*1024px",
                "value": "1024x1024"
            }, {
                "label": "1792px*1024px",
                "value": "1792x1024"
            }, {
                "label": "1024px*1792px",
                "value": "1024x1792"
            }
     ]

})
watch(()=>f.value.model,(n)=>{
    f.value.size='1024x1024';

    // 清空midjourney相关设置
    if (!n.startsWith('midjourney')) {
        f.value.bili = -1;
        f.value.quality = '';
        f.value.view = '';
        f.value.light = '';
        f.value.shot = '';
        f.value.style = '';
        f.value.styles = '';
        // f.value.version = '';
        f.value.sref = '';
        f.value.cref = '';
        f.value.cw = '';
    }
});

// 添加createPrompt方法处理midjourney提示词
function createPrompt(rz:string){
    if(rz =='') {
        return '';
    }

    let rzp='' // 参数组合字符串
    let rzk='' // 描述词组合字符串

    for(let v of farr){
        if(!f.value[v.k] || f.value[v.k]==null || f.value[v.k]=='') continue;

        if(v.k=='quality') rzp +=`  --q ${f.value.quality}`;
        else if(v.k=='styles') { if(f.value.styles) rzp +=` ${f.value.styles} `;}
        else if(v.k=='version') {
            if(['MID_JOURNEY','NIJI_JOURNEY'].indexOf(f.value.version)>-1){
                // 处理特殊版本
            } else rzp +=` ${f.value.version}`;
        }
        else if(f.value[v.k]) rzk +=`${f.value[v.k]},`;
    }

    if(f.value.sref.trim() != '') rzp += ` --sref ${f.value.sref}`
    if(f.value.cref.trim() != '') rzp += ` --cref ${f.value.cref}`
    if(f.value.cw && f.value.cw!='') rzp += ` --cw ${f.value.cw}`
    if(f.value.bili > -1) rzp += ` --ar ${vf[f.value.bili].label}`

    rz = rzk + rz + rzp;
    return rz;
}

// 保存设置到localStorage
watch(()=>f.value,(n)=>{
    localStorage.setItem("mjinput", JSON.stringify(n))
},{deep:true});

// 从localStorage加载设置
onMounted(()=>{
    let minput = localStorage.getItem('mjinput')
    if(minput){
      try {
        const a = JSON.parse(minput)
        // 只更新midjourney相关设置
        if(f.value.model.startsWith('midjourney')) {
            f.value.bili = a.bili || -1;
            f.value.quality = a.quality || '';
            f.value.view = a.view || '';
            f.value.light = a.light || '';
            f.value.shot = a.shot || '';
            f.value.style = a.style || '';
            f.value.styles = a.styles || '';
            // f.value.version = a.version || '';
            f.value.sref = a.sref || '';
            f.value.cref = a.cref || '';
            f.value.cw = a.cw || '';
        }
      } catch (error) {
        mlog("错误", error)
      }
    }
})
</script>
<template>
<section class="mb-4 flex justify-between items-center"  >
     <div>{{ $t('mjchat.version') }} </div>
    <n-select v-model:value="f.model" :options="modelConfig.model" filterable tag size="small"  class="!w-[70%]" :clearable="false" />
</section>
<section class="mb-4 flex justify-between items-center"  >
     <div>{{ $t('mjchat.size') }}</div>
    <n-select v-model:value="f.size" :options="dimensionsList"  filterable tag size="small"  class="!w-[70%]" :clearable="false" />
</section>

<!-- 当选择midjourney模型时显示的选项 -->
<template v-if="f.model.startsWith('mid')">
    <section class="mb-4">
        <div class="mr-1 mb-2 flex justify-between items-center">
            <div class="text-sm">{{ $t('mjchat.imgBili') }}</div>
        </div>
        <div class="flex items-center justify-between space-x-1">
            <template v-for="(item,index) in vf">
                <section class="aspect-item flex-1 rounded border-2 dark:border-neutral-700 cursor-pointer" :class="{'active':index==f.bili}" @click="f.bili=index">
                    <div class="aspect-box-wrapper mx-auto my-2 flex h-5 w-5 items-center justify-center">
                        <div class="aspect-box rounded border-2 dark:border-neutral-700" :style="item.s"></div>
                    </div>
                    <p class="mb-1 text-center text-sm">{{ item.label }}</p>
                </section>
            </template>
        </div>
    </section>

    <section class="mb-4 flex justify-between items-center" v-for="v in farr">
        <div>{{ v.v }}</div>
        <n-select v-model:value="f[v.k]" :options="drawlocalized[v.k+'List']" size="small" class="!w-[70%]" :clearable="true" />
    </section>

    <section class="mb-4 flex justify-between items-center">
        <div>cw(0-100)</div>
        <NInputNumber :min="0" :max="100" v-model:value="f.cw" class="!w-[70%]" size="small" clearable placeholder="0-100 角色参考程度" />
    </section>

    <section class="mb-4 flex justify-between items-center">
        <div class="w-[45px]">sref</div>
        <NInput v-model:value="f.sref" size="small" placeholder="图片url 生成风格一致的图像" clearable />
    </section>

    <section class="mb-4 flex justify-between items-center">
        <div class="w-[45px]">cref</div>
        <NInput v-model:value="f.cref" size="small" placeholder="图片url 生成角色一致的图像" clearable />
    </section>
</template>



<div class="mb-1">
     <n-input type="textarea" v-model:value="f.prompt" :placeholder="$t('mjchat.prompt')" round clearable maxlength="2000" show-count
      :autosize="{ minRows:3, maxRows:10 }" />
</div>

<div class="mb-4 flex justify-end items-center">
    <div class="flex ">
         <n-button type="primary" :block="true" :disabled="isDisabled" @click="create()"  >
            <SvgIcon icon="mingcute:send-plane-fill" />
             {{ $t('mjchat.imgcreate') }}
        </n-button>
    </div>
</div>

<ul class="pt-4" v-html="$t('mjchat.dalleInfo')">

</ul>
</template>
