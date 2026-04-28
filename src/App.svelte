<script>
  const STORAGE_KEY = 'stock-reporter-ai-config';

  const defaultConfig = {
    baseUrl: '',
    apiKey: '',
    model: ''
  };

  const sampleTickers = ['AAPL', 'MSFT', 'NVDA', 'TSLA', '005930.KS'];

  let config = loadConfig();
  let ticker = 'AAPL';
  let market = 'NASDAQ';
  let horizon = '중기';
  let language = '한국어';
  let riskTone = '균형';
  let models = [];
  let isLoadingModels = false;
  let isGenerating = false;
  let connectionStatus = '';
  let errorMessage = '';
  let report = '';

  $: canGenerate = config.baseUrl.trim() && config.model && ticker.trim() && !isGenerating;

  function loadConfig() {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      return saved ? { ...defaultConfig, ...JSON.parse(saved) } : { ...defaultConfig };
    } catch {
      return { ...defaultConfig };
    }
  }

  function saveConfig() {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(config));
  }

  function normalizeBaseUrl(value) {
    return value.trim().replace(/\/+$/, '');
  }

  function endpoint(path) {
    const base = normalizeBaseUrl(config.baseUrl);
    return `${base}${path.startsWith('/') ? path : `/${path}`}`;
  }

  function requestHeaders() {
    const headers = {
      'Content-Type': 'application/json'
    };

    if (config.apiKey.trim()) {
      headers.Authorization = `Bearer ${config.apiKey.trim()}`;
    }

    return headers;
  }

  async function loadModels() {
    errorMessage = '';
    connectionStatus = '';
    models = [];

    if (!config.baseUrl.trim()) {
      errorMessage = 'AI API 서버 주소를 입력해주세요.';
      return;
    }

    isLoadingModels = true;

    try {
      const response = await fetch(endpoint('/models'), {
        method: 'GET',
        headers: requestHeaders()
      });

      if (!response.ok) {
        throw new Error(`모델 목록 요청 실패: HTTP ${response.status}`);
      }

      const payload = await response.json();
      const list = Array.isArray(payload.data) ? payload.data : Array.isArray(payload.models) ? payload.models : [];
      models = list
        .map((model) => (typeof model === 'string' ? model : model.id || model.name))
        .filter(Boolean)
        .sort((a, b) => a.localeCompare(b));

      if (!models.length) {
        throw new Error('서버 응답에서 모델 목록을 찾지 못했습니다.');
      }

      if (!models.includes(config.model)) {
        config.model = models[0];
      }

      saveConfig();
      connectionStatus = `${models.length}개 모델을 불러왔습니다.`;
    } catch (error) {
      errorMessage = error.message || '모델 목록을 불러오지 못했습니다.';
    } finally {
      isLoadingModels = false;
    }
  }

  function buildPrompt() {
    return [
      '당신은 전문 주식 리서치 애널리스트입니다.',
      `종목: ${ticker.trim().toUpperCase()}`,
      `시장/거래소: ${market.trim() || '미지정'}`,
      `투자 관점: ${horizon}`,
      `위험 선호: ${riskTone}`,
      `작성 언어: ${language}`,
      '',
      '다음 형식으로 투자 참고용 리포트를 작성하세요.',
      '1. 핵심 요약',
      '2. 시장 및 업종 흐름',
      '3. 종목 강점과 리스크',
      '4. 확인해야 할 지표',
      '5. 투자 관점별 시나리오',
      '',
      '실시간 시세나 최신 재무 데이터가 서버 프롬프트에 제공되지 않았다면, 확정적인 최신 수치 대신 확인이 필요한 항목을 명확히 구분하세요. 투자 조언이 아니라 참고 리서치임을 유지하세요.'
    ].join('\n');
  }

  async function generateReport() {
    if (!canGenerate) return;

    errorMessage = '';
    report = '';
    isGenerating = true;

    try {
      const response = await fetch(endpoint('/chat/completions'), {
        method: 'POST',
        headers: requestHeaders(),
        body: JSON.stringify({
          model: config.model,
          messages: [
            {
              role: 'system',
              content: 'You write concise, well-structured stock research reports for retail investors. Avoid fabricating live market data.'
            },
            {
              role: 'user',
              content: buildPrompt()
            }
          ],
          temperature: 0.35
        })
      });

      if (!response.ok) {
        throw new Error(`리포트 생성 실패: HTTP ${response.status}`);
      }

      const payload = await response.json();
      report = payload.choices?.[0]?.message?.content || payload.output_text || payload.content || '';

      if (!report) {
        throw new Error('서버 응답에서 리포트 내용을 찾지 못했습니다.');
      }

      saveConfig();
    } catch (error) {
      errorMessage = error.message || '리포트를 생성하지 못했습니다.';
    } finally {
      isGenerating = false;
    }
  }

  function chooseTicker(value) {
    ticker = value;
  }
</script>

<svelte:head>
  <meta
    name="description"
    content="AI API 서버를 연결해 주식 분석 리포트를 생성하는 Svelte 웹 애플리케이션"
  />
</svelte:head>

<main class="app-shell">
  <section class="workspace">
    <div class="masthead">
      <div>
        <p class="eyebrow">AI Stock Research</p>
        <h1>StockReporter</h1>
      </div>
      <div class="market-visual" aria-hidden="true">
        <svg viewBox="0 0 260 120" role="img">
          <defs>
            <linearGradient id="line" x1="0" x2="1" y1="0" y2="0">
              <stop offset="0%" stop-color="#16a085" />
              <stop offset="55%" stop-color="#2364aa" />
              <stop offset="100%" stop-color="#d65a31" />
            </linearGradient>
          </defs>
          <path class="grid" d="M0 24H260M0 60H260M0 96H260" />
          <path class="chart-fill" d="M6 96L36 72L63 80L94 42L126 54L158 30L191 48L224 22L254 34V116H6Z" />
          <path class="chart-line" d="M6 96L36 72L63 80L94 42L126 54L158 30L191 48L224 22L254 34" />
        </svg>
      </div>
    </div>

    <div class="layout">
      <section class="panel controls" aria-label="리포트 설정">
        <div class="section-title">
          <h2>AI 서버</h2>
          <button class="ghost-button" type="button" on:click={loadModels} disabled={isLoadingModels}>
            {isLoadingModels ? '불러오는 중' : '모델 불러오기'}
          </button>
        </div>

        <label>
          <span>API 서버 주소</span>
          <input
            bind:value={config.baseUrl}
            on:change={saveConfig}
            placeholder="http://localhost:8000/v1"
            autocomplete="url"
          />
        </label>

        <label>
          <span>API Key</span>
          <input
            bind:value={config.apiKey}
            on:change={saveConfig}
            type="password"
            placeholder="선택 사항"
            autocomplete="off"
          />
        </label>

        <label>
          <span>모델</span>
          <select bind:value={config.model} on:change={saveConfig} disabled={!models.length}>
            {#if models.length}
              {#each models as model}
                <option value={model}>{model}</option>
              {/each}
            {:else}
              <option value="">먼저 모델을 불러오세요</option>
            {/if}
          </select>
        </label>

        {#if connectionStatus}
          <p class="status success">{connectionStatus}</p>
        {/if}

        <div class="divider"></div>

        <div class="section-title">
          <h2>리포트 입력</h2>
        </div>

        <div class="ticker-row" aria-label="빠른 종목 선택">
          {#each sampleTickers as item}
            <button
              class:active={ticker.toUpperCase() === item.toUpperCase()}
              type="button"
              on:click={() => chooseTicker(item)}
            >
              {item}
            </button>
          {/each}
        </div>

        <div class="field-grid">
          <label>
            <span>종목 코드</span>
            <input bind:value={ticker} placeholder="AAPL" />
          </label>

          <label>
            <span>시장</span>
            <input bind:value={market} placeholder="NASDAQ" />
          </label>
        </div>

        <div class="field-grid">
          <label>
            <span>투자 관점</span>
            <select bind:value={horizon}>
              <option>단기</option>
              <option>중기</option>
              <option>장기</option>
            </select>
          </label>

          <label>
            <span>위험 선호</span>
            <select bind:value={riskTone}>
              <option>보수적</option>
              <option>균형</option>
              <option>공격적</option>
            </select>
          </label>
        </div>

        <label>
          <span>작성 언어</span>
          <select bind:value={language}>
            <option>한국어</option>
            <option>English</option>
            <option>日本語</option>
          </select>
        </label>

        <button class="primary-button" type="button" on:click={generateReport} disabled={!canGenerate}>
          {isGenerating ? '리포트 생성 중' : '리포트 생성'}
        </button>

        {#if errorMessage}
          <p class="status error">{errorMessage}</p>
        {/if}
      </section>

      <section class="panel report-panel" aria-label="생성된 리포트">
        <div class="report-header">
          <div>
            <p class="eyebrow">Generated Report</p>
            <h2>{ticker.trim() ? ticker.trim().toUpperCase() : '종목'} 분석</h2>
          </div>
          <span>{config.model || '모델 미선택'}</span>
        </div>

        {#if report}
          <article class="report-content">
            {#each report.split('\n') as line}
              {#if line.trim()}
                <p>{line}</p>
              {:else}
                <br />
              {/if}
            {/each}
          </article>
        {:else}
          <div class="empty-state">
            <strong>리포트가 여기에 표시됩니다.</strong>
            <p>AI 서버 주소를 입력하고 모델을 불러온 뒤 종목을 선택해 생성하세요.</p>
          </div>
        {/if}
      </section>
    </div>
  </section>
</main>

