---
title: moments
date: 2025-11-11
layout: page
---

<div class="moments-timeline" id="moments-container">
  <p class="loading-state">加载中...</p>
</div>

<script>
(function() {
  // 从内联 script 标签读取数据
  let momentsData;

  // 从 /api 目录加载 JSON 文件
  fetch('/api/moments.json')
    .then(response => response.json())
    .then(data => {
      momentsData = data;
      renderMoments();
    })
    .catch(error => {
      // Fallback: 显示空状态
      console.error('加载时刻数据失败:', error);
      momentsData = {
        "moments": [],
        "lastSync": "",
        "totalCount": 0
      };
      renderMoments();
    });

  function renderMoments() {
    const container = document.getElementById('moments-container');

    if (!momentsData.moments || momentsData.moments.length === 0) {
      container.innerHTML = '<p class="empty-state">还没有记录任何时刻</p>';
      return;
    }

    // 渲染时刻记录
    let html = '';
    momentsData.moments.forEach(function(moment) {
      const createdAt = new Date(moment.createdAt);
      const updatedAt = new Date(moment.updatedAt);
      const isEdited = createdAt.getTime() !== updatedAt.getTime();

      html += '<article class="moment-item">';
      html += '<div class="moment-meta">';
      html += '<time datetime="' + moment.createdAt + '">';
      html += createdAt.toLocaleString('zh-CN');
      html += '</time>';
      if (isEdited) {
        html += '<span class="moment-edited">（已编辑）</span>';
      }
      html += '</div>';
      html += '<div class="moment-content">';
      html += renderMarkdown(moment.content);
      html += '</div>';
      html += '</article>';
    });

    container.innerHTML = html;
  }

  // 简单的 Markdown 渲染器
  function renderMarkdown(text) {
    return text
      .split('\n\n').map(function(para) {
        // 列表
        if (para.match(/^- /m)) {
          const items = para.split('\n').filter(function(line) { return line.startsWith('- '); })
            .map(function(line) { return '<li>' + renderInline(line.substring(2)) + '</li>'; })
            .join('');
          return '<ul>' + items + '</ul>';
        }
        // 普通段落
        return '<p>' + renderInline(para.replace(/\n/g, '<br>')) + '</p>';
      }).join('');
  }

  function renderInline(text) {
    return text
      .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
      .replace(/\*(.+?)\*/g, '<em>$1</em>')
      .replace(/`(.+?)`/g, '<code>$1</code>')
      .replace(/\[(.+?)\]\((.+?)\)/g, '<a href="$2" target="_blank">$1</a>');
  }
})();
</script>

<style>
.moments-timeline {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem 0;
}

.moment-item {
  margin-bottom: 2rem;
  padding: 1.5rem;
  border-left: 3px solid #3498db;
  background: #f8f9fa;
  border-radius: 4px;
  transition: transform 0.2s, box-shadow 0.2s;
}

.moment-item:hover {
  transform: translateX(4px);
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.moment-meta {
  font-size: 0.9rem;
  color: #6c757d;
  margin-bottom: 0.75rem;
}

.moment-edited {
  margin-left: 0.5rem;
  font-style: italic;
  color: #95a5a6;
}

.moment-content {
  line-height: 1.8;
  word-break: break-word;
}

.moment-content p:last-child {
  margin-bottom: 0;
}

.moment-content ul {
  margin: 0.5rem 0;
  padding-left: 1.5rem;
}

.moment-content code {
  background: #e8e8e8;
  padding: 0.2em 0.4em;
  border-radius: 3px;
  font-size: 0.9em;
}

.moment-content a {
  color: #3498db;
  text-decoration: none;
}

.moment-content a:hover {
  text-decoration: underline;
}

.empty-state, .loading-state {
  text-align: center;
  color: #6c757d;
  padding: 3rem 1rem;
  font-size: 1.1rem;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .moment-item {
    padding: 1rem;
    margin-bottom: 1.5rem;
  }

  .moments-timeline {
    padding: 1rem;
  }
}
</style>
