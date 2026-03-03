<template>
  <div class="practice-container">
    <!-- 顶部导航 -->
    <nav class="practice-nav animate__animated animate__fadeIn">
      <div class="nav-brand">
        <span class="brand-icon">📝</span>
        <div class="brand-text">
          <div class="main-title">练习模式</div>
          <div class="sub-title">基于知识库的智能出题与评分</div>
        </div>
      </div>
      <div class="actions">
        <select v-model="selectedKnowledge" class="knowledge-select" @change="onKnowledgeChange">
          <option value="">随机知识库</option>
          <option v-for="kb in knowledgeList" :key="kb.id" :value="kb.id">
            {{ kb.name }}
          </option>
        </select>
        <button
          class="btn-generate"
          @click="generateQuestions"
          :disabled="isGenerating"
          :class="{ 'is-loading': isGenerating }"
        >
          <span v-if="isGenerating" class="loading-icon"></span>
          <span>{{ isGenerating ? '正在生成题目...' : '生成题目' }}</span>
        </button>
      </div>
    </nav>

    <!-- 主要内容区 -->
    <div class="main-content">
      <!-- 左侧：题目列表 -->
      <section class="questions-section">
        <div class="section-header">
          <span class="section-title">题目列表</span>
          <span class="question-count" v-if="questions.length > 0">
            共 {{ questions.length }} 道题目
          </span>
        </div>

        <div class="questions-list" v-if="questions.length > 0">
          <div
            v-for="(q, index) in questions"
            :key="index"
            class="question-card"
            :class="{ 'active': currentQuestionIndex === index }"
            @click="currentQuestionIndex = index"
          >
            <div class="question-header">
              <span class="question-number">第 {{ index + 1 }} 题</span>
              <span class="question-type" :class="q.type">
                {{ q.type === 'choice' ? '选择题' : q.type === 'multi' ? '多选题' : q.type === 'judge' ? '判断题' : q.type === 'code' ? '代码题' : q.type === 'short' ? '简答题' : '填空题' }}
              </span>
              <!-- 用空心圆/实心圆表示作答状态 -->
              <span class="answer-status" :class="{ 'answered': q.userAnswer }">
                {{ q.userAnswer ? '●' : '○' }}
              </span>
            </div>
            <div class="question-preview">{{ q.content.substring(0, 50) }}...</div>
          </div>
        </div>

        <div class="empty-state" v-else>
          <div class="empty-icon">📚</div>
          <p>点击上方"生成题目"开始练习</p>
        </div>
      </section>

      <!-- 右侧：答题区 -->
      <aside class="answer-section">
        <Transition name="fade-slide" mode="out-in">
          <!-- 有题目时显示答题区 -->
          <div v-if="currentQuestion && !showResult" class="answer-panel" key="answer">
            <div class="answer-header">
              <span class="current-label">
                第 {{ currentQuestionIndex + 1 }} / {{ questions.length }} 题
              </span>
              <span class="question-type-badge" :class="currentQuestion.type">
                {{ currentQuestion.type === 'choice' ? '选择题' : currentQuestion.type === 'multi' ? '多选题' : currentQuestion.type === 'judge' ? '判断题' : currentQuestion.type === 'code' ? '代码题' : currentQuestion.type === 'short' ? '简答题' : '填空题' }}
              </span>
            </div>

            <div class="question-content">
              <p>{{ currentQuestion.content }}</p>
            </div>

            <!-- 选择题选项 -->
            <div class="options-list" v-if="currentQuestion.type === 'choice'">
              <label
                v-for="(option, idx) in currentQuestion.options"
                :key="idx"
                class="option-item"
                :class="{ 'selected': currentQuestion.userAnswer === getOptionLetter(option, idx) }"
                @click="selectOption(option, idx)"
              >
                <span class="option-letter">{{ getOptionLetter(option, idx) }}</span>
                <span class="option-text">{{ getOptionContent(option) }}</span>
              </label>
            </div>

            <!-- 多选题选项 -->
            <div class="options-list multi" v-else-if="currentQuestion.type === 'multi'">
              <p class="multi-hint">（多选题，可选择多个答案）</p>
              <label
                v-for="(option, idx) in currentQuestion.options"
                :key="idx"
                class="option-item"
                :class="{ 'selected': isMultiSelected(option, idx) }"
                @click="toggleMultiOption(option, idx)"
              >
                <span class="option-checkbox">{{ isMultiSelected(option, idx) ? '☑' : '☐' }}</span>
                <span class="option-letter">{{ getOptionLetter(option, idx) }}</span>
                <span class="option-text">{{ getOptionContent(option) }}</span>
              </label>
            </div>

            <!-- 判断题选项 -->
            <div class="judge-options" v-else-if="currentQuestion.type === 'judge'">
              <label
                class="judge-item"
                :class="{ 'selected': currentQuestion.userAnswer === '对' }"
                @click="currentQuestion.userAnswer = '对'"
              >
                <span class="judge-icon">✓</span>
                <span class="judge-text">正确</span>
              </label>
              <label
                class="judge-item"
                :class="{ 'selected': currentQuestion.userAnswer === '错' }"
                @click="currentQuestion.userAnswer = '错'"
              >
                <span class="judge-icon">✗</span>
                <span class="judge-text">错误</span>
              </label>
            </div>

            <!-- 代码题输入 -->
            <div class="code-input" v-else-if="currentQuestion.type === 'code'">
              <textarea
                v-model="currentQuestion.userAnswer"
                placeholder="请在此输入你的代码..."
                rows="12"
                class="code-textarea"
              ></textarea>
            </div>

            <!-- 简答题输入 -->
            <div class="short-answer" v-else-if="currentQuestion.type === 'short'">
              <p class="short-hint">（简答题，需要老师人工评分）</p>
              <textarea
                v-model="currentQuestion.userAnswer"
                placeholder="请输入你的答案..."
                rows="6"
              ></textarea>
            </div>

            <!-- 填空题输入 -->
            <div class="fill-blank" v-else>
              <textarea
                v-model="currentQuestion.userAnswer"
                placeholder="请输入你的答案..."
                rows="4"
              ></textarea>
            </div>

            <!-- 导航按钮 -->
            <div class="nav-buttons">
              <button
                class="btn-nav"
                @click="prevQuestion"
                :disabled="currentQuestionIndex === 0"
              >
                上一题
              </button>
              <button
                class="btn-nav"
                @click="nextQuestion"
                :disabled="currentQuestionIndex === questions.length - 1"
              >
                下一题
              </button>
            </div>

            <!-- 提交按钮 -->
            <button
              class="btn-submit-all"
              @click="submitAllAnswers"
              :disabled="!canSubmit || isSubmitting"
              :class="{ 'is-loading': isSubmitting }"
            >
              <span v-if="isSubmitting" class="loading-icon"></span>
              <span>{{ isSubmitting ? gradingProgress || 'AI 正在评分中...' : '提交全部答案' }}</span>
            </button>
            
            <!-- 评分进度提示 -->
            <div v-if="isSubmitting && gradingProgress" class="grading-progress">
              <div class="progress-bar">
                <div class="progress-fill" :style="{ width: (questions.filter(q => q.isCorrect !== null).length / questions.length * 100) + '%' }"></div>
              </div>
              <p class="progress-text">{{ gradingProgress }}，请稍候...</p>
            </div>
          </div>

          <!-- 结果展示 -->
          <div v-else-if="showResult" class="result-panel" key="result">
            <div class="result-header">
              <span class="ai-badge">评分报告</span>
              <button class="close-btn" @click="resetQuiz">重新开始</button>
            </div>

            <div class="score-summary">
              <div class="score-circle" :style="{ borderColor: getScoreColor(totalScore) }">
                <span class="score">{{ totalScore }}</span>
                <span class="unit">分</span>
              </div>
              <div class="score-info">
                <p>正确: {{ correctCount }} / {{ questions.length }}</p>
                <p>正确率: {{ Math.round(correctCount / questions.length * 100) }}%</p>
              </div>
            </div>

            <!-- 详细结果 -->
            <div class="detailed-results">
              <div
                v-for="(q, index) in questions"
                :key="index"
                class="result-item"
                :class="{ 'correct': q.isCorrect, 'wrong': !q.isCorrect }"
              >
                <div class="result-question">
                  <span class="result-number">第 {{ index + 1 }} 题</span>
                  <span class="result-status" v-if="q.type === 'code' && q.score !== undefined">
                    <span :class="getScoreClass(q.score)">{{ getScoreLevel(q.score) }}</span>
                    <span class="code-score">({{ q.score }}分)</span>
                  </span>
                  <span class="result-status" v-else>{{ q.isCorrect ? '✓ 正确' : '✗ 错误' }}</span>
                </div>
                <div class="result-content">
                  <p class="result-q">{{ q.content }}</p>
                  <p class="result-answer">
                    <span class="label">你的答案:</span>
                    <span :class="{ 'wrong-text': !q.isCorrect }">{{ formatAnswer(q.userAnswer) || '未作答' }}</span>
                  </p>
                  <p class="result-correct" v-if="!q.isCorrect && q.type !== 'code' && q.type !== 'short'">
                    <span class="label">正确答案:</span>
                    <span class="correct-text">{{ formatAnswer(q.correctAnswer) }}</span>
                  </p>
                  <p class="result-correct" v-if="q.type === 'short'">
                    <span class="label">参考答案:</span>
                    <span class="correct-text">{{ formatAnswer(q.correctAnswer) }}</span>
                    <span class="manual-hint">（需老师人工评分）</span>
                  </p>
                  <p class="result-correct" v-if="q.type === 'code' && q.correctAnswer">
                    <span class="label">参考答案:</span>
                    <span class="correct-text">{{ q.correctAnswer }}</span>
                  </p>
                  <div class="result-explanation" v-if="q.explanation">
                    <span class="label">解析:</span>
                    <p>{{ q.explanation }}</p>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- 无题目时的引导 -->
          <div v-else class="guide-section" key="guide">
            <h3>📚 练习模式说明</h3>
            <div class="guide-card">
              <h4>如何开始？</h4>
              <ul class="guide-list">
                <li>1. 选择知识库（默认随机抽取）</li>
                <li>2. 点击"生成题目"获取10-15道练习题</li>
                <li>3. 完成答题后提交获取评分</li>
                <li>4. 查看解析，巩固知识</li>
              </ul>
            </div>
            <div class="tip-box">
              <i class="icon-info">💡</i>
              <p>题目类型包括选择题和填空题，内容来自所选知识库的文档。</p>
            </div>
          </div>
        </Transition>
      </aside>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';

// 状态
const knowledgeList = ref([]);
const selectedKnowledge = ref('');
const questions = ref([]);
const currentQuestionIndex = ref(0);
const isGenerating = ref(false);
const isSubmitting = ref(false);
const showResult = ref(false);
const totalScore = ref(0);
const correctCount = ref(0);
const gradingProgress = ref(''); // 评分进度提示

// MaxKB API 配置
const MAXKB_BASE_URL = 'http://localhost:8090';
const MAXKB_TOKEN = 'Bearer application-bb3982ea3ad34a8264a5ac2f7ce2b78b';

// 获取选项字母 (支持 A-Z)
const getOptionLetter = (option, index) => {
  if (!option) return '';
  // 匹配开头的字母
  const match = option.match(/^([A-Za-z])[\.\、\s]/);
  if (match) {
    return match[1].toUpperCase();
  }
  // 如果没有字母前缀，根据索引返回
  const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  return letters[index] || 'A';
};

// 获取选项内容（去掉字母前缀）
const getOptionContent = (option) => {
  if (!option) return '';
  // 去掉开头的字母和分隔符
  return option.replace(/^[A-Za-z][\.\、\s]?\s*/, '');
};

// 格式化答案显示（处理数组格式）
const formatAnswer = (answer) => {
  if (!answer) return '';
  if (Array.isArray(answer)) {
    return answer.join(', ');
  }
  return String(answer);
};

// 选择选项
const selectOption = (option, index) => {
  const letter = getOptionLetter(option, index);
  // 如果已选择同一选项，取消选择；否则选择新选项
  if (currentQuestion.value.userAnswer === letter) {
    currentQuestion.value.userAnswer = '';
  } else {
    currentQuestion.value.userAnswer = letter;
  }
};

// 多选题：检查是否选中
const isMultiSelected = (option, index) => {
  const letter = getOptionLetter(option, index);
  const userAnswer = currentQuestion.value.userAnswer || [];
  return Array.isArray(userAnswer) && userAnswer.includes(letter);
};

// 多选题：切换选项
const toggleMultiOption = (option, index) => {
  const letter = getOptionLetter(option, index);
  if (!Array.isArray(currentQuestion.value.userAnswer)) {
    currentQuestion.value.userAnswer = [];
  }
  const idx = currentQuestion.value.userAnswer.indexOf(letter);
  if (idx > -1) {
    currentQuestion.value.userAnswer.splice(idx, 1);
  } else {
    currentQuestion.value.userAnswer.push(letter);
  }
  // 排序答案
  currentQuestion.value.userAnswer.sort();
};

// 当前题目
const currentQuestion = computed(() => {
  return questions.value[currentQuestionIndex.value] || null;
});

// 是否可以提交
const canSubmit = computed(() => {
  return questions.value.some(q => q.userAnswer);
});

// 知识库切换
const onKnowledgeChange = () => {
  console.log('切换知识库:', selectedKnowledge.value);
};

// 获取知识库列表
const fetchKnowledgeList = async () => {
  try {
    // 尝试不同的 API 路径
    let response = await fetch(`${MAXKB_BASE_URL}/api/knowledge/workspace/default/knowledge`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': MAXKB_TOKEN
      }
    });
    
    // 如果失败，尝试另一个路径
    if (!response.ok) {
      response = await fetch(`${MAXKB_BASE_URL}/api/application/workspace/default/application`, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': MAXKB_TOKEN
        }
      });
    }
    
    if (response.ok) {
      const text = await response.text();
      try {
        const data = JSON.parse(text);
        knowledgeList.value = data.data || [];
      } catch (e) {
        console.log('知识库列表解析失败，使用默认模式');
        knowledgeList.value = [];
      }
    }
  } catch (error) {
    console.log('获取知识库列表失败，将使用随机模式:', error.message);
    knowledgeList.value = [];
  }
};

// 生成题目
const generateQuestions = async () => {
  isGenerating.value = true;
  questions.value = [];
  currentQuestionIndex.value = 0;
  showResult.value = false;

  try {
    // 获取会话 ID
    console.log('正在获取会话ID...');
    const chatIdResponse = await fetch(`${MAXKB_BASE_URL}/chat/api/open`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': MAXKB_TOKEN
      }
    });

    if (!chatIdResponse.ok) {
      throw new Error(`获取会话ID失败: ${chatIdResponse.status}`);
    }
    
    const chatIdData = await chatIdResponse.json();
    console.log('会话ID响应:', chatIdData);
    const chatId = chatIdData.data || chatIdData.id || '';
    
    if (!chatId) {
      throw new Error('无法获取有效的会话ID');
    }
    console.log('获取到的会话ID:', chatId);

    // 构造出题 prompt - 随机生成 10-15 道题
    const questionCount = Math.floor(Math.random() * 6) + 10; // 10-15 道随机

    const prompt = `请根据知识库文档内容生成${questionCount}道练习题。

【强制要求】
1. 只返回纯JSON格式，不要任何其他文字、解释或Markdown格式
2. 不要使用\`\`\`json或\`\`\`代码块标记
3. 直接以 {"questions": 开头，以 } 结尾
4. 题目必须来自知识库内容

返回格式示例：
{"questions":[{"type":"choice","content":"题目内容?","options":["A. 选项1","B. 选项2","C. 选项3","D. 选项4"],"correctAnswer":"A","explanation":"解析"},{"type":"fill","content":"填空题____","correctAnswer":"答案","explanation":"解析"},{"type":"judge","content":"判断题内容","correctAnswer":"对","explanation":"解析"},{"type":"code","content":"代码题要求","correctAnswer":"参考答案","explanation":"评分标准"}]}

现在直接返回JSON：`;

    // 发送出题请求
    const response = await fetch(`${MAXKB_BASE_URL}/chat/api/chat_message/${chatId}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': MAXKB_TOKEN
      },
      body: JSON.stringify({
        message: prompt,
        stream: false,
        re_chat: true
      })
    });

    const data = await response.json();
    console.log('MaxKB 完整响应:', data);
    
    // 检查是否返回错误
    if (data.code !== 200) {
      if (data.message && data.message.includes('访问次数')) {
        throw new Error('MaxKB 访问次数超限，请稍后再试');
      }
      throw new Error(data.message || 'MaxKB 返回错误');
    }
    
    const aiReply = data.data?.content || data.content || data.answer || '';
    console.log('提取的回复内容:', aiReply);
    
    if (!aiReply) {
      throw new Error('MaxKB 返回内容为空');
    }
    
    // 解析题目
    parseQuestions(aiReply);

  } catch (error) {
    console.error('生成题目失败:', error);
    alert('生成题目失败: ' + error.message);
  } finally {
    isGenerating.value = false;
  }
};

// 解析 AI 返回的题目
const parseQuestions = (text) => {
  console.log('MaxKB 返回原始内容:', text);
  
  try {
    // 预处理：清理各种干扰字符
    let cleanedText = text;
    
    // 移除 markdown 代码块标记
    cleanedText = cleanedText.replace(/```json\s*/gi, '');
    cleanedText = cleanedText.replace(/```\s*/g, '');
    
    // 移除 "复制代码" 等文字
    cleanedText = cleanedText.replace(/复制代码/g, '');
    
    // 【关键修复】替换中文标点为英文标点
    cleanedText = cleanedText.replace(/"/g, '"');
    cleanedText = cleanedText.replace(/"/g, '"');
    cleanedText = cleanedText.replace(/'/g, "'");
    cleanedText = cleanedText.replace(/'/g, "'");
    cleanedText = cleanedText.replace(/，/g, ',');
    cleanedText = cleanedText.replace(/：/g, ':');
    cleanedText = cleanedText.replace(/【/g, '[');
    cleanedText = cleanedText.replace(/】/g, ']');
    
    // 【修复】无效转义字符
    cleanedText = cleanedText.replace(/\\(?!["\\\/bfnrtu])/g, '\\\\');
    
    // 【修复】移除不可见字符
    cleanedText = cleanedText.replace(/[\u200B-\u200D\uFEFF\u00A0]/g, '');
    
    // 【关键修复】处理代码块中的换行符
    // 把 ```代码块``` 里面的换行符替换成 \n 转义序列
    cleanedText = cleanedText.replace(/```(\w*)\n([\s\S]*?)```/g, (match, lang, code) => {
      // 把代码里的换行符转义
      const escapedCode = code.replace(/\n/g, '\\n').replace(/\r/g, '\\r').replace(/\t/g, '\\t');
      return '```' + lang + '\\n' + escapedCode + '```';
    });
    
    console.log('清理后内容长度:', cleanedText.length);
    
    // 【新方法】提取所有 JSON 对象
    const allQuestions = [];
    let searchStart = 0;
    
    while (searchStart < cleanedText.length) {
      // 找下一个 { 
      const braceIdx = cleanedText.indexOf('{', searchStart);
      if (braceIdx === -1) break;
      
      // 使用状态机找到匹配的结束大括号
      let braceCount = 0;
      let inString = false;
      let escapeNext = false;
      let endIdx = -1;
      
      for (let i = braceIdx; i < cleanedText.length; i++) {
        const char = cleanedText[i];
        
        if (escapeNext) {
          escapeNext = false;
          continue;
        }
        
        if (char === '\\') {
          escapeNext = true;
          continue;
        }
        
        if (char === '"') {
          inString = !inString;
          continue;
        }
        
        if (!inString) {
          if (char === '{') {
            braceCount++;
          } else if (char === '}') {
            braceCount--;
            if (braceCount === 0) {
              endIdx = i + 1;
              break;
            }
          }
        }
      }
      
      if (endIdx === -1) {
        searchStart = braceIdx + 1;
        continue;
      }
      
      // 提取这个 JSON 字符串
      let jsonStr = cleanedText.substring(braceIdx, endIdx);
      
      // 调试：打印原始 JSON 片段
      console.log('原始 JSON (前200字符):', jsonStr.substring(0, 200));
      
      // 【关键修复】修复JSON字符串值里的问题
      // 1. 换行符需要转义
      // 2. 字符串内部的双引号需要转义
      let fixedJson = '';
      let inJsonString = false;
      let jsonEscape = false;
      
      for (let i = 0; i < jsonStr.length; i++) {
        const char = jsonStr[i];
        
        if (jsonEscape) {
          fixedJson += char;
          jsonEscape = false;
          continue;
        }
        
        if (char === '\\' && inJsonString) {
          fixedJson += char;
          jsonEscape = true;
          continue;
        }
        
        // 处理字符串外的反斜杠（可能是错误的转义）
        if (char === '\\' && !inJsonString) {
          // 跳过字符串外的孤立反斜杠
          continue;
        }
        
        if (char === '"') {
          if (!inJsonString) {
            // 开始一个字符串
            inJsonString = true;
            fixedJson += char;
          } else {
            // 检查这个引号是字符串结束还是内部引号
            // 如果后面紧跟的是 : , } ] 或空白+这些字符，说明是字符串结束
            const rest = jsonStr.substring(i + 1).trimStart();
            const nextChar = rest.charAt(0);
            if ([':', ',', '}', ']', '\n', '\r', ''].includes(nextChar) || rest === '') {
              inJsonString = false;
              fixedJson += char;
            } else {
              // 这是字符串内部的引号，需要转义
              fixedJson += '\\"';
            }
          }
          continue;
        }
        
        // 在字符串里面遇到换行符，转义它
        if (inJsonString) {
          if (char === '\n') {
            fixedJson += '\\n';
          } else if (char === '\r') {
            fixedJson += '\\r';
          } else if (char === '\t') {
            fixedJson += '\\t';
          } else {
            fixedJson += char;
          }
        } else {
          fixedJson += char;
        }
      }
      
      // 尝试解析
      try {
        // 【修复1】移除尾随逗号（数组/对象最后一个元素后的逗号）
        let repairJson = fixedJson
          .replace(/,\s*\]/g, ']')   // 数组尾随逗号
          .replace(/,\s*\}/g, '}');  // 对象尾随逗号
        
        // 调试：打印修复后的 JSON 片段
        console.log('修复后 JSON (前300字符):', repairJson.substring(0, 300));
        
        let parsed;
        try {
          parsed = JSON.parse(repairJson);
        } catch (parseError) {
          // 【修复2】尝试添加缺失的逗号（保守方式）
          // 情况：`"correctAnswer": "A"\n"explanation"` 缺少逗号
          repairJson = repairJson.replace(/"\s*\n\s*"/g, '",\n"');
          console.log('添加逗号后 JSON (前300字符):', repairJson.substring(0, 300));
          parsed = JSON.parse(repairJson);
        }
        
        // 如果有 questions 数组，提取里面的题目
        if (parsed.questions && Array.isArray(parsed.questions)) {
          allQuestions.push(...parsed.questions);
        } 
        // 如果是单个题目对象（有 type 字段）
        else if (parsed.type && parsed.content) {
          allQuestions.push(parsed);
        }
        
        console.log('解析到一个 JSON，当前题目总数:', allQuestions.length);
      } catch (e) {
        console.log('JSON 解析失败，跳过:', e.message.substring(0, 50));
      }
      
      searchStart = endIdx;
    }
    
    if (allQuestions.length === 0) {
      // 【备用】尝试从 Markdown 格式解析
      console.log('JSON 解析失败，尝试 Markdown 解析...');
      const markdownQuestions = parseMarkdownQuestions(cleanedText);
      if (markdownQuestions.length > 0) {
        questions.value = markdownQuestions;
        console.log('成功从 Markdown 解析题目:', questions.value.length, '道');
        return;
      }
      
      throw new Error('未找到有效的题目数据');
    }
    
    // 处理所有题目
    questions.value = allQuestions.map(q => {
      // 【修复】标准化题型名称
      let type = q.type || 'fill';
      
      // 映射各种题型名称
      const typeMap = {
        'choose': 'choice',
        'true/false': 'judge',
        'calc': 'fill',       // 计算题 → 填空题
        'sort': 'fill',       // 排序题 → 填空题
        'graph': 'judge',     // 图形题 → 判断题
        'function': 'code',   // 函数题 → 代码题
      };
      
      if (typeMap[type]) {
        type = typeMap[type];
      }
      
      // 【关键修复】智能检测题型
      const hasOptions = Array.isArray(q.options) && q.options.length >= 2;
      const correctAnswer = q.correctAnswer || q.correct || '';
      
      // 检测多选题：correctAnswer 是数组
      if (hasOptions && Array.isArray(correctAnswer) && correctAnswer.length > 0) {
        type = 'multi';
      }
      // 检测选择题：correctAnswer 是单个字母
      else if (hasOptions && typeof correctAnswer === 'string') {
        const answerLetter = correctAnswer.toUpperCase();
        if (/^[A-E]$/.test(answerLetter)) {
          type = 'choice';
        }
      }
      // 检测简答题：答案是很长的文本
      else if (type === 'short' || (typeof correctAnswer === 'string' && correctAnswer.length > 50)) {
        type = 'short';
      }
      
      // 处理选项（选择题和多选题）
      let options = [];
      if ((type === 'choice' || type === 'multi') && hasOptions) {
        const letters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
        options = q.options.map((opt, idx) => {
          if (typeof opt === 'string' && !opt.match(/^[A-Za-z][\.\、\s]/)) {
            return `${letters[idx]}. ${opt}`;
          }
          return opt;
        });
      }
      
      // 标准化答案格式
      let normalizedAnswer = correctAnswer;
      if (typeof normalizedAnswer === 'string' && normalizedAnswer.length === 1) {
        normalizedAnswer = normalizedAnswer.toUpperCase();
      }
      // 多选题答案转大写数组
      if (Array.isArray(normalizedAnswer)) {
        normalizedAnswer = normalizedAnswer.map(a => typeof a === 'string' ? a.toUpperCase() : a);
      }
      
      return {
        ...q,
        type,
        options,
        correctAnswer: normalizedAnswer,
        userAnswer: type === 'multi' ? [] : '',
        isCorrect: null
      };
    });
    
    console.log('成功解析题目:', questions.value.length, '道');
    
  } catch (e) {
    console.error('解析题目异常:', e);
    alert('题目生成失败: ' + e.message);
  }
};

// 从 Markdown 格式解析题目（备用方案）
const parseMarkdownQuestions = (text) => {
  const questions = [];
  console.log('尝试 Markdown 解析...');
  
  // 【检测问答题格式】如果是问答题格式（长答案），直接返回空
  const isQAType = text.includes('### 基础题') ||
                   text.includes('### 进阶题') ||
                   text.includes('### 答案参考') ||
                   text.includes('### 总结');
  
  if (isQAType) {
    console.log('检测到问答题格式，不支持解析');
    return [];
  }
  
  // ========== 选择题 ==========
  // 格式1: **选择题**：题目内容 后面跟选项和答案
  // 格式2: 数字. 题目内容 后面跟选项和答案
  
  // 【格式1】- **选择题**：题目内容
  const choiceFormat1 = /[-*]\s*\*\*选择题\*\*[：:]\s*([^\n]+)\n\s*([A-E])\.\s*([^\n]+)\n\s*([A-E])\.\s*([^\n]+)\n\s*([A-E])\.\s*([^\n]+)(?:\n\s*([A-E])\.\s*([^\n]+))?(?:\n\s*([A-E])\.\s*([^\n]+))?(?:\n\s*\*\*正确答案[：:]\s*([A-E])\*\*)?/g;
  let m1;
  while ((m1 = choiceFormat1.exec(text)) !== null) {
    const content = m1[1].trim();
    if (content.length < 5 || content.includes('答案')) continue;
    
    const options = [];
    const letters = [m1[2], m1[4], m1[6]];
    const contents = [m1[3], m1[5], m1[7]];
    if (m1[8] && m1[9]) { letters.push(m1[8]); contents.push(m1[9]); }
    if (m1[10] && m1[11]) { letters.push(m1[10]); contents.push(m1[11]); }
    
    for (let i = 0; i < letters.length; i++) {
      options.push(`${letters[i]}. ${contents[i].trim()}`);
    }
    
    const correctAnswer = m1[12] || 'A';
    questions.push({
      type: 'choice',
      content,
      options,
      correctAnswer,
      explanation: '',
      userAnswer: '',
      isCorrect: null
    });
  }
  
  // 【格式2】数字. 题目内容 后面跟选项和 **正确答案**
  const choiceFormat2 = /(\d+)\.\s*([^\n]+)\n\s*([A-E])\.\s*([^\n]+)\n\s*([A-E])\.\s*([^\n]+)\n\s*([A-E])\.\s*([^\n]+)(?:\n\s*([A-E])\.\s*([^\n]+))?(?:\n\s*([A-E])\.\s*([^\n]+))?\n\s*\*\*正确答案[：:]\s*([A-E])\*\*/g;
  let m2;
  while ((m2 = choiceFormat2.exec(text)) !== null) {
    const content = m2[2].trim();
    if (content.length < 5 || content.includes('答案')) continue;
    
    const options = [
      `${m2[3]}. ${m2[4].trim()}`,
      `${m2[5]}. ${m2[6].trim()}`,
      `${m2[7]}. ${m2[8].trim()}`
    ];
    if (m2[9] && m2[10]) options.push(`${m2[9]}. ${m2[10].trim()}`);
    if (m2[11] && m2[12]) options.push(`${m2[11]}. ${m2[12].trim()}`);
    
    questions.push({
      type: 'choice',
      content,
      options,
      correctAnswer: m2[13],
      explanation: '',
      userAnswer: '',
      isCorrect: null
    });
  }
  
  // ========== 填空题 ==========
  // 格式: **填空题**：题目内容________答案
  const fillPattern = /[-*]\s*\*\*填空题\*\*[：:]\s*([^\n]*)(?:_+|_{2,})([^\n]*)/g;
  let fillMatch;
  while ((fillMatch = fillPattern.exec(text)) !== null) {
    const content = fillMatch[1].trim();
    const answer = fillMatch[2].trim();
    if (content.length > 5 && answer.length > 0 && answer.length < 50) {
      questions.push({
        type: 'fill',
        content: content + ' ____',
        correctAnswer: answer,
        explanation: '',
        userAnswer: '',
        isCorrect: null
      });
    }
  }
  
  // ========== 判断题 ==========
  // 格式1: **判断题**：题目内容。（对/错）或 (√/×)
  // 格式2: 题目内容 **正确答案：对/错**
  
  const judgeFormat1 = /[-*]\s*\*\*判断题\*\*[：:]\s*([^\n]+?)(?:（(对|错|√|×)）|(?:\*\*正确答案[：:]\s*(对|错)\*\*))?/g;
  let jm1;
  while ((jm1 = judgeFormat1.exec(text)) !== null) {
    let content = jm1[1].trim();
    // 清理末尾的符号
    content = content.replace(/[（(][√×对错][）)]$/g, '').trim();
    if (content.length > 5 && !content.includes('答案')) {
      // 如果有明确答案就用，否则默认"对"
      const answer = jm1[2] || jm1[3] || '对';
      const normalizedAnswer = ['√', '对', '是', 'true'].includes(answer) ? '对' : '错';
      questions.push({
        type: 'judge',
        content,
        correctAnswer: normalizedAnswer,
        explanation: '',
        userAnswer: '',
        isCorrect: null
      });
    }
  }
  
  // 格式2: 数字. 题目内容 **正确答案：对/错**
  const judgeFormat2 = /(\d+)\.\s*([^\n]+?)\s*\*\*正确答案[：:]\s*(对|错)\*\*/g;
  let jm2;
  while ((jm2 = judgeFormat2.exec(text)) !== null) {
    const content = jm2[2].trim();
    if (content.length > 5 && !content.includes('答案')) {
      questions.push({
        type: 'judge',
        content,
        correctAnswer: jm2[3],
        explanation: '',
        userAnswer: '',
        isCorrect: null
      });
    }
  }
  
  // ========== 代码题 ==========
  const codePattern = /[-*]\s*\*\*代码题\*\*[：:]\s*([^\n]+)/g;
  let codeMatch;
  while ((codeMatch = codePattern.exec(text)) !== null) {
    const content = codeMatch[1].trim();
    if (content.length > 5) {
      questions.push({
        type: 'code',
        content,
        correctAnswer: '参考代码实现',
        explanation: '',
        userAnswer: '',
        isCorrect: null
      });
    }
  }
  
  console.log('Markdown 解析结果:', questions.length, '道有效题目');
  return questions;
};

// 提交全部答案 - 选择题/填空题/判断题本地评分，代码题调用 MaxKB 评分
const submitAllAnswers = async () => {
  isSubmitting.value = true;
  gradingProgress.value = '正在评分...';

  let score = 0;
  let correct = 0;
  const totalQuestions = questions.value.length;
  const pointsPerQuestion = Math.round(100 / totalQuestions);

  // 先收集代码题，需要调用 MaxKB 评分
  const codeQuestions = questions.value.filter(q => q.type === 'code' && q.userAnswer && q.userAnswer.trim());
  
  // 如果有代码题，获取会话 ID
  let chatId = '';
  if (codeQuestions.length > 0) {
    try {
      const chatIdResponse = await fetch(`${MAXKB_BASE_URL}/chat/api/open`, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': MAXKB_TOKEN
        }
      });
      const chatIdData = await chatIdResponse.json();
      chatId = chatIdData.data || chatIdData.id || '';
    } catch (e) {
      console.error('获取会话ID失败:', e);
    }
  }

  for (let i = 0; i < questions.value.length; i++) {
    const q = questions.value[i];
    
    // 显示评分进度
    gradingProgress.value = `正在评分: ${i + 1}/${totalQuestions} 题`;
    
    // 模拟短暂延迟，让用户看到进度
    await new Promise(resolve => setTimeout(resolve, 50));
    
    if (!q.userAnswer || !String(q.userAnswer).trim()) {
      q.isCorrect = false;
      q.score = 0;
      continue;
    }

    // 检查 correctAnswer 是否存在
    const hasCorrectAnswer = Array.isArray(q.correctAnswer) 
      ? q.correctAnswer.length > 0 
      : q.correctAnswer && String(q.correctAnswer).trim();
    
    if (!hasCorrectAnswer) {
      console.warn('题目缺少正确答案:', q.content);
      q.isCorrect = false;
      q.score = 0;
      continue;
    }

    // 【修复】确保答案是字符串类型
    const correctAnswerStr = Array.isArray(q.correctAnswer) 
      ? q.correctAnswer.join(',') 
      : String(q.correctAnswer || '');
    
    const userAnswerStr = Array.isArray(q.userAnswer)
      ? q.userAnswer.join(',')
      : String(q.userAnswer || '');
    
    const userAnswerUpper = userAnswerStr.toUpperCase().trim();
    const correctAnswerUpper = correctAnswerStr.toUpperCase().trim();
    
    if (q.type === 'choice') {
      // 选择题：只比较选项字母
      q.isCorrect = userAnswerUpper === correctAnswerUpper;
      q.score = q.isCorrect ? 100 : 0;
    } else if (q.type === 'multi') {
      // 多选题：数组完全匹配
      const userArr = userAnswerStr.split(',').map(s => s.trim()).sort();
      const correctArr = correctAnswerStr.split(',').map(s => s.trim()).sort();
      q.isCorrect = JSON.stringify(userArr) === JSON.stringify(correctArr);
      // 多选题部分正确给部分分
      const matched = userArr.filter(a => correctArr.includes(a)).length;
      if (q.isCorrect) {
        q.score = 100;
      } else if (matched > 0 && userArr.length <= correctArr.length) {
        q.score = Math.round((matched / correctArr.length) * 100);
      } else {
        q.score = 0;
      }
    } else if (q.type === 'judge') {
      // 判断题：标准化答案比较
      // 正确答案的可能格式: "对", "是", "true", "正确"
      // 错误答案的可能格式: "错", "否", "false", "错误"
      const normalizeJudgeAnswer = (ans) => {
        const a = ans.toLowerCase().trim();
        if (['对', '是', 'true', '正确', 'yes', '√', '✓'].includes(a)) return '对';
        if (['错', '否', 'false', '错误', 'no', '×', '✗'].includes(a)) return '错';
        return a;
      };
      const normalizedUser = normalizeJudgeAnswer(userAnswerUpper);
      const normalizedCorrect = normalizeJudgeAnswer(correctAnswerUpper);
      q.isCorrect = normalizedUser === normalizedCorrect;
      q.score = q.isCorrect ? 100 : 0;
    } else if (q.type === 'code') {
      // 代码题：调用 MaxKB 评分
      gradingProgress.value = `正在评分: ${i + 1}/${totalQuestions} 题 (代码题评分中...)`;
      const result = await gradeCodeQuestion(chatId, q);
      q.isCorrect = result.isCorrect;
      // 代码题按实际得分比例计算
      score += Math.round(pointsPerQuestion * (result.score / 100));
      if (q.isCorrect) correct++;
      continue; // 跳过下面的统分逻辑
    } else if (q.type === 'short') {
      // 简答题：需要人工评分，暂时给0分并标记
      q.isCorrect = false;
      q.score = 0;
      q.needsManualGrading = true;
    } else {
      // 填空题：模糊匹配
      q.isCorrect = userAnswerUpper === correctAnswerUpper || 
                   userAnswerUpper.includes(correctAnswerUpper) ||
                   correctAnswerUpper.includes(userAnswerUpper);
      q.score = q.isCorrect ? 100 : 0;
    }
    
    if (q.isCorrect) {
      score += pointsPerQuestion;
      correct++;
    }
  }

  // 确保总分不超过 100
  totalScore.value = Math.min(score, 100);
  correctCount.value = correct;
  showResult.value = true;
  gradingProgress.value = '';
  isSubmitting.value = false;
};

// 代码题评分 - 调用 MaxKB
const gradeCodeQuestion = async (chatId, question) => {
  if (!chatId) {
    // 无法调用 MaxKB，默认给 60% 分
    return { isCorrect: false, score: 0 };
  }

  const prompt = `你是一位专业的代码评审专家。请评判以下代码：

题目要求：${question.content}

参考答案/评分标准：
${question.correctAnswer}

用户提交的代码：
${question.userAnswer}

请根据以下标准评分：
- 优秀(90-100分)：代码完全正确，逻辑清晰，风格规范
- 良好(80-89分)：代码基本正确，有小瑕疵但不影响功能
- 及格(60-79分)：代码思路正确，但有明显错误或不完整
- 不及格(0-59分)：代码错误较多或完全错误

请只返回JSON格式：
{
  "score": 0-100的分数,
  "level": "优秀/良好/及格/不及格",
  "explanation": "简短的评分说明（100字以内）"
}`;

  try {
    const response = await fetch(`${MAXKB_BASE_URL}/chat/api/chat_message/${chatId}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': MAXKB_TOKEN
      },
      body: JSON.stringify({
        message: prompt,
        stream: false,
        re_chat: true
      })
    });

    const data = await response.json();
    const aiReply = data.data?.content || '';
    console.log('代码题评分原始回复:', aiReply);
    
    // 解析 AI 返回的 JSON（复用修复逻辑）
    let jsonStr = aiReply;
    
    // 提取 JSON 部分
    const jsonMatch = jsonStr.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      console.log('未找到 JSON 格式');
      return { isCorrect: false, score: 0 };
    }
    
    jsonStr = jsonMatch[0];
    
    // 修复 JSON 字符串值里的问题
    let fixedJson = '';
    let inJsonString = false;
    let jsonEscape = false;
    
    for (let i = 0; i < jsonStr.length; i++) {
      const char = jsonStr[i];
      
      if (jsonEscape) {
        fixedJson += char;
        jsonEscape = false;
        continue;
      }
      
      if (char === '\\' && inJsonString) {
        fixedJson += char;
        jsonEscape = true;
        continue;
      }
      
      if (char === '"') {
        if (!inJsonString) {
          inJsonString = true;
          fixedJson += char;
        } else {
          const rest = jsonStr.substring(i + 1).trimStart();
          const nextChar = rest.charAt(0);
          if ([':', ',', '}', ']', '\n', '\r', ''].includes(nextChar) || rest === '') {
            inJsonString = false;
            fixedJson += char;
          } else {
            fixedJson += '\\"';
          }
        }
        continue;
      }
      
      if (inJsonString) {
        if (char === '\n') {
          fixedJson += '\\n';
        } else if (char === '\r') {
          fixedJson += '\\r';
        } else if (char === '\t') {
          fixedJson += '\\t';
        } else {
          fixedJson += char;
        }
      } else {
        fixedJson += char;
      }
    }
    
    // 尝试解析修复后的 JSON
    try {
      const parsed = JSON.parse(fixedJson);
      // 更新解析内容
      if (parsed.explanation) {
        question.explanation = parsed.explanation;
      }
      if (parsed.level) {
        question.level = parsed.level;
      }
      const score = parsed.score || 0;
      question.score = score;
      // 分数 >= 60 算正确
      return { isCorrect: score >= 60, score };
    } catch (parseError) {
      console.log('JSON 解析仍失败，尝试提取数字:', parseError.message);
      // 尝试直接提取分数数字
      const scoreMatch = aiReply.match(/"score"\s*:\s*(\d+)/);
      if (scoreMatch) {
        const score = parseInt(scoreMatch[1]);
        question.score = score;
        return { isCorrect: score >= 60, score };
      }
    }
  } catch (e) {
    console.error('代码题评分失败:', e);
  }
  
  return { isCorrect: false, score: 0 };
};

// 重置
const resetQuiz = () => {
  questions.value = [];
  currentQuestionIndex.value = 0;
  showResult.value = false;
  totalScore.value = 0;
  correctCount.value = 0;
};

// 上一题
const prevQuestion = () => {
  if (currentQuestionIndex.value > 0) {
    currentQuestionIndex.value--;
  }
};

// 下一题
const nextQuestion = () => {
  if (currentQuestionIndex.value < questions.value.length - 1) {
    currentQuestionIndex.value++;
  }
};

// 分数颜色
const getScoreColor = (s) => {
  if (s >= 80) return '#22c55e';
  if (s >= 60) return '#eab308';
  return '#ef4444';
};

// 评分等级
const getScoreLevel = (s) => {
  if (s >= 90) return '优秀';
  if (s >= 80) return '良好';
  if (s >= 60) return '及格';
  return '不及格';
};

// 评分等级样式类
const getScoreClass = (s) => {
  if (s >= 90) return 'level-excellent';
  if (s >= 80) return 'level-good';
  if (s >= 60) return 'level-pass';
  return 'level-fail';
};

onMounted(() => {
  fetchKnowledgeList();
});
</script>

<style scoped>
.practice-container {
  height: 100%;
  display: flex;
  flex-direction: column;
  background: #0d1117;
  color: #c9d1d9;
}

.practice-nav {
  height: 64px;
  background: #161b22;
  border-bottom: 1px solid #30363d;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 24px;
}

.nav-brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.brand-icon {
  font-size: 1.8rem;
}

.main-title {
  font-size: 1.1rem;
  font-weight: bold;
  color: #58a6ff;
}

.sub-title {
  font-size: 0.7rem;
  color: #8b949e;
}

.actions {
  display: flex;
  gap: 12px;
  align-items: center;
}

.knowledge-select {
  background: #21262d;
  color: #c9d1d9;
  border: 1px solid #30363d;
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  min-width: 150px;
}

.knowledge-select:hover {
  border-color: #58a6ff;
}

.btn-generate {
  background: #238636;
  color: white;
  border: none;
  padding: 10px 24px;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: background 0.2s;
}

.btn-generate:hover:not(:disabled) {
  background: #2ea043;
}

.btn-generate:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.btn-generate.is-loading {
  opacity: 0.7;
  cursor: wait;
}

.loading-icon {
  width: 16px;
  height: 16px;
  border: 2px solid #fff;
  border-top-color: transparent;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.main-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

/* 左侧题目列表 */
.questions-section {
  width: 320px;
  background: #161b22;
  border-right: 1px solid #30363d;
  display: flex;
  flex-direction: column;
}

.section-header {
  padding: 16px 20px;
  border-bottom: 1px solid #30363d;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.section-title {
  font-weight: 600;
  color: #58a6ff;
}

.question-count {
  font-size: 0.8rem;
  color: #8b949e;
}

.questions-list {
  flex: 1;
  overflow-y: auto;
  padding: 12px;
}

.question-card {
  background: #0d1117;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 10px;
  cursor: pointer;
  transition: all 0.2s;
}

.question-card:hover {
  border-color: #58a6ff;
}

.question-card.active {
  border-color: #58a6ff;
  background: #161b22;
}

.question-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.question-number {
  font-size: 0.85rem;
  font-weight: 600;
  color: #c9d1d9;
}

.question-type {
  font-size: 0.7rem;
  padding: 2px 8px;
  border-radius: 4px;
}

.question-type.choice {
  background: #1f3d5c;
  color: #58a6ff;
}

.question-type.fill {
  background: #3d1f5c;
  color: #a371f7;
}

.question-type.judge {
  background: #5c3d1f;
  color: #f0883e;
}

.question-type.code {
  background: #1f5c3d;
  color: #3fb950;
}

/* 作答状态 - 空心圆/实心圆 */
.answer-status {
  margin-left: auto;
  font-size: 0.9rem;
  color: #6e7681;
}

.answer-status.answered {
  color: #58a6ff;
}

.question-preview {
  font-size: 0.8rem;
  color: #8b949e;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.empty-state {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #8b949e;
}

.empty-icon {
  font-size: 3rem;
  margin-bottom: 16px;
}

/* 右侧答题区 */
.answer-section {
  flex: 1;
  background: #0d1117;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
}

.answer-panel {
  padding: 24px;
}

.answer-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.current-label {
  font-size: 1.1rem;
  font-weight: 600;
  color: #58a6ff;
}

.question-type-badge {
  font-size: 0.8rem;
  padding: 4px 12px;
  border-radius: 4px;
}

.question-type-badge.choice {
  background: #1f3d5c;
  color: #58a6ff;
}

.question-type-badge.fill {
  background: #3d1f5c;
  color: #a371f7;
}

.question-type-badge.judge {
  background: #5c3d1f;
  color: #f0883e;
}

.question-type-badge.code {
  background: #1f5c3d;
  color: #3fb950;
}

.question-content {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 24px;
  font-size: 1rem;
  line-height: 1.6;
}

.options-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
}

.option-item {
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 14px 16px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  gap: 12px;
}

.option-item:hover {
  border-color: #58a6ff;
  background: #1f2d3a;
}

.option-item.selected {
  border-color: #58a6ff;
  background: #1f3d5c;
}

.option-item input {
  display: none;
}

.option-letter {
  width: 28px;
  height: 28px;
  background: #21262d;
  border: 2px solid #30363d;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 0.9rem;
  color: #8b949e;
  flex-shrink: 0;
  transition: all 0.2s;
}

.option-item.selected .option-letter {
  background: #58a6ff;
  border-color: #58a6ff;
  color: white;
}

.option-item:hover .option-letter {
  border-color: #58a6ff;
}

.option-text {
  font-size: 0.95rem;
  flex: 1;
}

.fill-blank textarea {
  width: 100%;
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 16px;
  color: #c9d1d9;
  font-size: 0.95rem;
  resize: none;
  outline: none;
}

.fill-blank textarea:focus {
  border-color: #58a6ff;
}

/* 多选题样式 */
.options-list.multi {
  border: 1px dashed #30363d;
  border-radius: 8px;
  padding: 8px;
}

.multi-hint {
  color: #8b949e;
  font-size: 0.85rem;
  margin-bottom: 12px;
  padding-left: 8px;
}

.option-checkbox {
  font-size: 1.2rem;
  color: #8b949e;
  margin-right: 4px;
}

.option-item.selected .option-checkbox {
  color: #58a6ff;
}

/* 简答题样式 */
.short-answer {
  position: relative;
}

.short-hint {
  color: #f0883e;
  font-size: 0.85rem;
  margin-bottom: 12px;
  padding: 8px 12px;
  background: rgba(240, 136, 62, 0.1);
  border-radius: 6px;
  border-left: 3px solid #f0883e;
}

.short-answer textarea {
  width: 100%;
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 16px;
  color: #c9d1d9;
  font-size: 0.95rem;
  resize: none;
  outline: none;
}

.short-answer textarea:focus {
  border-color: #58a6ff;
}

.manual-hint {
  color: #f0883e;
  font-size: 0.85rem;
  margin-left: 8px;
}

/* 判断题样式 */
.judge-options {
  display: flex;
  gap: 16px;
  margin-bottom: 24px;
}

.judge-item {
  flex: 1;
  background: #161b22;
  border: 2px solid #30363d;
  border-radius: 12px;
  padding: 24px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.judge-item:hover {
  border-color: #58a6ff;
  background: #1f2d3a;
}

.judge-item.selected {
  border-color: #58a6ff;
  background: #1f3d5c;
}

.judge-icon {
  font-size: 2.5rem;
  font-weight: bold;
}

.judge-item.selected .judge-icon {
  color: #58a6ff;
}

.judge-text {
  font-size: 1.1rem;
  font-weight: 500;
}

/* 代码题样式 */
.code-input {
  margin-bottom: 24px;
}

.code-textarea {
  width: 100%;
  background: #0d1117;
  border: 1px solid #30363d;
  border-radius: 8px;
  padding: 16px;
  color: #e6edf3;
  font-family: 'Fira Code', 'Consolas', monospace;
  font-size: 0.9rem;
  line-height: 1.6;
  resize: vertical;
  outline: none;
}

.code-textarea:focus {
  border-color: #58a6ff;
}

.nav-buttons {
  display: flex;
  gap: 12px;
  margin-bottom: 24px;
}

.btn-nav {
  flex: 1;
  background: #21262d;
  color: #c9d1d9;
  border: 1px solid #30363d;
  padding: 10px 20px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-nav:hover:not(:disabled) {
  background: #30363d;
}

.btn-nav:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-submit-all {
  width: 100%;
  background: #238636;
  color: white;
  border: none;
  padding: 14px;
  border-radius: 6px;
  font-weight: 600;
  font-size: 1rem;
  cursor: pointer;
  transition: background 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.btn-submit-all:hover:not(:disabled) {
  background: #2ea043;
}

.btn-submit-all:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-submit-all.is-loading {
  opacity: 0.7;
  cursor: wait;
}

/* 结果面板 */
.result-panel {
  padding: 24px;
}

.result-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.ai-badge {
  background: #1f3d5c;
  color: #58a6ff;
  padding: 6px 16px;
  border-radius: 4px;
  font-size: 0.85rem;
  font-weight: 600;
}

.close-btn {
  background: transparent;
  color: #8b949e;
  border: 1px solid #30363d;
  padding: 6px 16px;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.close-btn:hover {
  background: #21262d;
  color: #c9d1d9;
}

.score-summary {
  display: flex;
  align-items: center;
  gap: 32px;
  margin-bottom: 32px;
  padding: 24px;
  background: #161b22;
  border-radius: 12px;
}

.score-circle {
  width: 100px;
  height: 100px;
  border: 5px solid #238636;
  border-radius: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.score {
  font-size: 2.2rem;
  font-weight: bold;
}

.unit {
  font-size: 0.8rem;
  color: #8b949e;
}

.score-info {
  font-size: 0.95rem;
  color: #8b949e;
}

.score-info p {
  margin: 4px 0;
}

.detailed-results {
  display: flex;
  flex-direction: column;
  gap: 16px;
  max-height: 400px;
  overflow-y: auto;
}

.result-item {
  background: #161b22;
  border-radius: 8px;
  padding: 16px;
  border-left: 4px solid #30363d;
}

.result-item.correct {
  border-left-color: #238636;
}

.result-item.wrong {
  border-left-color: #f85149;
}

.result-question {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.result-number {
  font-weight: 600;
  color: #c9d1d9;
}

.result-status {
  font-size: 0.85rem;
}

.result-item.correct .result-status {
  color: #238636;
}

.result-item.wrong .result-status {
  color: #f85149;
}

/* 代码题评分等级样式 */
.level-excellent {
  color: #22c55e;
  font-weight: bold;
}

.level-good {
  color: #3b82f6;
  font-weight: bold;
}

.level-pass {
  color: #eab308;
  font-weight: bold;
}

.level-fail {
  color: #ef4444;
  font-weight: bold;
}

.code-score {
  color: #8b949e;
  font-size: 0.8rem;
  margin-left: 4px;
}

.result-content {
  font-size: 0.9rem;
}

.result-q {
  color: #c9d1d9;
  margin-bottom: 8px;
}

.result-answer, .result-correct {
  margin: 8px 0;
  color: #8b949e;
}

.label {
  color: #6e7681;
  margin-right: 8px;
}

.wrong-text {
  color: #f85149;
}

.correct-text {
  color: #238636;
}

.result-explanation {
  background: #0d1117;
  padding: 12px;
  border-radius: 6px;
  margin-top: 12px;
  color: #8b949e;
}

/* 引导区 */
.guide-section {
  padding: 24px;
}

.guide-section h3 {
  color: #58a6ff;
  margin-bottom: 20px;
}

.guide-card {
  background: #161b22;
  padding: 20px;
  border-radius: 8px;
  border: 1px solid #30363d;
  margin-bottom: 20px;
}

.guide-card h4 {
  color: #c9d1d9;
  margin-bottom: 12px;
}

.guide-list {
  list-style: none;
  padding: 0;
  margin: 0;
  color: #8b949e;
  font-size: 0.9rem;
}

.guide-list li {
  margin-bottom: 10px;
  padding-left: 8px;
}

.tip-box {
  background: #1f3d5c;
  padding: 16px;
  border-radius: 8px;
  display: flex;
  align-items: flex-start;
  gap: 12px;
}

.icon-info {
  font-size: 1.2rem;
}

.tip-box p {
  color: #8b949e;
  font-size: 0.9rem;
  margin: 0;
}

/* 动画 */
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.3s ease;
}

.fade-slide-enter-from {
  opacity: 0;
  transform: translateX(20px);
}

.fade-slide-leave-to {
  opacity: 0;
  transform: translateX(-20px);
}

/* 评分进度 */
.grading-progress {
  margin-top: 16px;
  padding: 16px;
  background: #161b22;
  border-radius: 8px;
  border: 1px solid #30363d;
}

.progress-bar {
  height: 8px;
  background: #21262d;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 12px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #238636, #2ea043);
  border-radius: 4px;
  transition: width 0.3s ease;
}

.progress-text {
  color: #8b949e;
  font-size: 0.85rem;
  text-align: center;
  margin: 0;
}
</style>
