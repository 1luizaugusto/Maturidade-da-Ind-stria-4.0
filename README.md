# RoupaAudio – Especificação Técnica

Aplicativo Android nativo projetado para pessoas cegas ou com baixa visão organizarem roupas usando códigos táteis (A000–D999), descrições textuais e audiodescrição. Todo o fluxo prioriza comandos de voz, feedback auditivo e alto contraste visual.

---

## 1. Visão Geral de Arquitetura

```
+---------------------------------------------------------------+
| Camada de Apresentação (Compose + ViewModels)                 |
|  - Navegação por voz e toques grandes                         |
|  - Integração com TTS/STT                                     |
+-------------------------------+-------------------------------+
| Camada de Domínio (Use Cases) | Serviços auxiliares           |
|  - ConsultarRoupa             | - TextToSpeechManager         |
|  - CadastrarRoupa             | - SpeechRecognizerManager     |
|  - ListarRoupasPorCategoria   | - BackupManager (GoogleDrive) |
+-------------------------------+-------------------------------+
| Camada de Dados (Room)                                        |
|  - RoupaDao / RoupaDatabase                                   |
|  - Repositório RoupaRepository                                |
+---------------------------------------------------------------+
| Persistência: Arquivos locais (áudio/foto) + Google Drive     |
+---------------------------------------------------------------+
```

- **MVVM** garante separação entre UI e lógica de negócio.
- **Room** fornece persistência offline-first.
- **Serviços de voz** encapsulam TTS/STT para reutilização e testes.
- **BackupManager** coordena exportação/importação com Google Drive, sem bloquear fluxo offline.

---

## 2. Modelo de Dados Room

```kotlin
data class Roupa(
    @PrimaryKey(autoGenerate = true) val id: Long = 0L,
    val codigo_letra: Char,              // 'A'..'D'
    val codigo_numero: String,           // "000".."999"
    val descricao_texto: String,
    val audio_path: String,
    val foto_uri: String? = null,
    val data_criacao: Long = System.currentTimeMillis()
)
```

```kotlin
@Dao
interface RoupaDao {
    @Insert suspend fun insert(roupa: Roupa): Long
    @Update suspend fun update(roupa: Roupa)
    @Delete suspend fun delete(roupa: Roupa)

    @Query("SELECT * FROM Roupa WHERE codigo_letra = :letra AND codigo_numero = :numero LIMIT 1")
    suspend fun findByCodigo(letra: Char, numero: String): Roupa?

    @Query("SELECT * FROM Roupa ORDER BY codigo_letra, codigo_numero")
    fun observeAll(): Flow<List<Roupa>>
}

@Database(entities = [Roupa::class], version = 1)
abstract class RoupaDatabase : RoomDatabase() {
    abstract fun roupaDao(): RoupaDao
}
```

**Regra de domínio:** código completo `A000–D999` deve ser único; validações ocorrem no caso de uso `CadastrarRoupa` e na tela de leitura.

---

## 3. Fluxos e Telas Principais

### Tela Inicial (Opções 1–3)
- **Entrada:** app saúda a pessoa, explica opções e ativa escuta para "Opção 1/2/3".
- **Confirmação:** após reconhecer comando, TTS pergunta "Confirmar?" e aguarda "sim"/"não".
- **Layout:** fundo preto, título branco, botões pílula amarelos ocupando largura total com labels acessíveis (TalkBack).

### Tela de Leitura de Código
- **Entrada:** TTS apresenta orientações, ativa STT para código (ex.: "A nove três cinco").
- **Teclado simplificado:** botões grandes para letras A–D, números 0–9, Apagar, Limpar, Confirmar.
- **Feedback:** cada toque ecoa o símbolo inserido; erro de formato dispara mensagem e reescuta.
- **Resultado encontrado:** TTS lê descrição, oferece ações (ouvir texto/áudio, editar, novo código, voltar, sair). Áudio gravado é reproduzido se `audio_path` existir.
- **Resultado inexistente:** TTS indica ausência e oferece cadastrar com o código, tentar outro, voltar ou sair.

### Tela de Cadastro de Roupa
1. **Código:** mesmo teclado/voz da leitura; verifica duplicidade. Se existir, pode editar ou escolher outro.
2. **Descrição:** STT captura frase; texto é exibido em Lexend grande. TTS confirma e permite regravar.
3. **Gravação:** botão amarelo "Gravar áudio" com instruções faladas; salva `.mp3` local. Oferece ouvir/regravar.
4. **Foto (opcional):** pergunta via voz; se sim, abre câmera e salva URI.
5. **Resumo:** TTS lista código, descrição, presença de áudio/foto e pergunta se salva. Ações extras permitem voltar a qualquer passo.

### Tela de Lista de Roupas
- **Agrupamento:** categorias A–D (Cabeça, Tórax, Pernas, Pés) com contagens anunciadas ao entrar.
- **Filtro:** comandos "Listar categoria B" ou botões de filtro.
- **Cards:** itens com código + descrição, botão de áudio e rótulos acessíveis. Toque ou voz "abrir A935" mostra ações: ouvir texto, ouvir áudio, editar, excluir, voltar.
- **Exclusão:** confirmação via voz antes de remover da base.

### Navegação Geral
- As telas exibem instruções textuais (alto contraste) e auditivas. Microfone ativa automaticamente após prompts, respeitando permissões e possibilidade de reativar via botão "Escutar novamente".

---

## 4. ViewModels e Use Cases

```kotlin
class MainViewModel(
    private val tts: TextToSpeechManager,
    private val stt: SpeechRecognizerManager
) : ViewModel() {
    val uiState = MutableStateFlow(MainUiState())

    fun onEnter() {
        tts.speak("Você está na tela inicial do RoupaAudio. ...")
        stt.listenForOptions(listOf("opção 1", "opção 2", "opção 3")) { option ->
            uiState.update { it.copy(pendingOption = option) }
            tts.askConfirmation(option) { confirmed -> /* navegar */ }
        }
    }
}

class LeituraViewModel(
    private val consultarRoupa: ConsultarRoupaUseCase,
    private val tts: TextToSpeechManager,
    private val stt: SpeechRecognizerManager
) : ViewModel() {
    val codigo = MutableStateFlow("")
    val roupaEncontrada = MutableStateFlow<Roupa?>(null)

    fun onVoiceInputRequested() {
        stt.listenForCodigo { codigoReconhecido ->
            codigo.value = codigoReconhecido
            confirmarCodigo()
        }
    }

    private fun confirmarCodigo() { /* TTS pergunta, consulta DAO via caso de uso */ }
}

class CadastroViewModel(/* deps */) : ViewModel() {
    val formState = MutableStateFlow(CadastroState())
    fun definirCodigo(codigo: String) { /* validações + TTS */ }
    fun salvarDescricao(texto: String) { /* eco */ }
    fun salvarAudio(path: String) { /* atualiza estado */ }
    fun confirmarCadastro() { /* chama use case e responde por voz */ }
}

class ListaViewModel(
    private val listarRoupas: ListarRoupasUseCase,
    private val deletarRoupa: DeletarRoupaUseCase
) : ViewModel() {
    val categorias = listarRoupas().stateIn(viewModelScope, SharingStarted.Eagerly, emptyMap())
    fun onDelete(roupa: Roupa) { /* confirma via TTS antes de chamar use case */ }
}
```

**Use cases** centralizam regras (ex.: `ConsultarRoupaUseCase`, `CadastrarRoupaUseCase`, `ListarPorCategoriaUseCase`, `EditarRoupaUseCase`, `ExcluirRoupaUseCase`, `BackupUseCase`). Cada um depende de `RoupaRepository` para facilitar testes.

---

## 5. Composables Principais (esqueleto)

```kotlin
@Composable
fun TelaInicial(uiState: MainUiState, onOptionSelected: (MainOption) -> Unit) {
    ScreenScaffold(title = "RoupaAudio") {
        Column(modifier = Modifier.fillMaxWidth().padding(24.dp)) {
            MainOption.values().forEach { option ->
                PillButton(
                    text = option.label,
                    onClick = { onOptionSelected(option) },
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(80.dp)
                        .semantics { contentDescription = option.accessibleLabel }
                )
                Spacer(Modifier.height(16.dp))
            }
        }
    }
}
```

```kotlin
@Composable
fun TelaLeitura(state: LeituraUiState, onInput: (CodigoInput) -> Unit, onAction: (LeituraAction) -> Unit) {
    ScreenScaffold(title = "Leitura de código", onBack = { onAction(LeituraAction.Voltar) }) {
        Text("Diga ou digite o código", style = Typography.h4, color = Color.White)
        CodigoDisplay(state.codigo)
        TecladoCodigo(onInput = onInput, onConfirm = { onAction(LeituraAction.Confirmar) })
        state.resultado?.let { roupa ->
            DescricaoCard(roupa)
            ActionButtons(roupa, onAction)
        }
    }
}
```

```kotlin
@Composable
fun TelaCadastro(state: CadastroState, onEvent: (CadastroEvent) -> Unit) { /* passos guiados */ }

@Composable
fun TelaLista(state: ListaUiState, onAction: (ListaAction) -> Unit) {
    LazyColumn {
        state.categorias.forEach { categoria ->
            item { CategoriaHeader(categoria) }
            items(categoria.itens) { roupa ->
                RoupaCard(roupa, onAction)
            }
        }
    }
}
```

`ScreenScaffold` aplica fundo preto, tipografia Lexend e TopAppBar com título centralizado. `PillButton` implementa o estilo amarelo com foco visível. Todos os composables usam `semantics { }` para TalkBack.

---

## 6. Integração com TTS/STT

```kotlin
class TextToSpeechManager(context: Context) : TextToSpeech.OnInitListener {
    private val tts = TextToSpeech(context, this)
    override fun onInit(status: Int) {
        if (status == TextToSpeech.SUCCESS) {
            tts.language = Locale("pt", "BR")
        }
    }

    fun speak(text: String, onDone: (() -> Unit)? = null) {
        val utteranceId = UUID.randomUUID().toString()
        if (onDone != null) {
            tts.setOnUtteranceProgressListener(object : UtteranceProgressListener() {
                override fun onDone(id: String?) { if (id == utteranceId) onDone() }
                override fun onError(id: String?) {}
                override fun onStart(id: String?) {}
            })
        }
        tts.speak(text, TextToSpeech.QUEUE_FLUSH, bundleOf(), utteranceId)
    }
}

class SpeechRecognizerManager(context: Context) {
    private val recognizer = SpeechRecognizer.createSpeechRecognizer(context)
    fun listenForCodigo(onResult: (String) -> Unit) {
        val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, "pt-BR")
        }
        recognizer.setRecognitionListener(object : RecognitionListenerAdapter() {
            override fun onResults(results: Bundle) {
                val spoken = results.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()
                onResult(formataCodigo(spoken))
            }
        })
        recognizer.startListening(intent)
    }
}
```

`formataCodigo` converte fala "A nove três cinco" para `A935`, validando letra e três dígitos. Caso inválido, TTS informa erro e `listenForCodigo` é reativado.

---

## 7. Backup e Restauração via Google Drive

1. **Autenticação:** usar Google Sign-In + Drive REST API com escopo `DRIVE_APPDATA`.
2. **Estrutura de arquivos:**
   - Pasta `RoupaAudioBackup` contendo
     - `database.db` (Room exportado via `getDatabasePath`).
     - Pasta `audios/` com `.mp3` (nomes baseados no ID da roupa).
     - Pasta `fotos/` com imagens associadas.
     - Manifest JSON com metadados e checksums.
3. **Backup flow:**
   - `BackupManager.startBackup()` fala "Backup iniciado", exporta DB + arquivos para zip, envia ao Drive, confirma via TTS.
4. **Restore flow:**
   - Lista backups disponíveis, confirma escolha por voz, baixa zip, substitui DB e arquivos locais, reinicia Room e informa sucesso/erro.
5. **Fallback offline:** se Drive indisponível, TTS orienta tentar novamente ou continuar offline.

---

## 8. Testes de Usabilidade com Pessoas Cegas

| Cenário | Objetivo | Métricas |
| --- | --- | --- |
| **Primeiro acesso** | Validar clareza das instruções iniciais e permissões de microfone. | Tempo para escolher opção correta; nº de repetições necessárias; percepção subjetiva de segurança. |
| **Leitura de código** | Avaliar precisão STT e ergonomia do teclado simplificado. | Taxa de sucesso na primeira tentativa; tempo médio até ouvir descrição; erros de navegação. |
| **Cadastro completo** | Medir fluidez do fluxo multi-etapas (código, descrição, áudio, foto). | Tempo total; nº de correções por etapa; satisfação verbal (Likert 1–5). |
| **Explorar lista** | Comandos de filtro, seleção por voz e exclusão com confirmação. | Nº de comandos mal interpretados; tempo para localizar item específico; confiança relatada. |
| **Backup Drive** | Compreender feedback e mensagens em caso de erro offline. | Tempo para concluir backup/restauração; incidência de dúvidas; avaliação qualitativa de clareza. |

**Processo:** realizar sessões com pelo menos 5 pessoas cegas, observar interações com TalkBack ativo, registrar dificuldades e ajustar prompts, tempos de TTS e tamanho dos elementos.

---

## 9. Próximos Passos

- Implementar protótipo navegável com dados mock para validar voz + UI.
- Integrar STT/TTS e medir desempenho em dispositivos reais.
- Adicionar testes instrumentados focados em acessibilidade (semantics, contraste) e cobertura de casos de uso.
