```
// Define a classe que representa um único item da lista
export class Item {
  private description: string; // Atributo privado para armazenar o texto da tarefa

  constructor(description: string) {
    this.description = description; // Inicializa a descrição ao criar o objeto
  }

  // Método para atualizar o texto de uma tarefa existente
  updateDescription(newDescription: string) {
    this.description = newDescription;
  }

  // Converte o objeto para um formato JSON simples (objeto literal)
  toJSON() {
    return {
      description: this.description
    }
  }
}

// Define a classe principal que gerencia a lista completa e o arquivo de salvamento
export class ToDo {
  private filepath: string; // Caminho do arquivo JSON onde os dados são salvos
  private items: Promise<Item[]>; // Promessa que resolverá no array de itens carregados

  constructor(filepath: string) {
    this.filepath = filepath; // Define o caminho do arquivo
    this.items = this.loadFromFile(); // Inicia o carregamento dos dados assim que a classe é instanciada
  }

  // Método privado para persistir os dados atuais no disco
  private async saveToFile() {
    try {
      const items = await this.items; // Aguarda a lista de itens estar pronta
      const file = Bun.file(this.filepath); // Referencia o arquivo usando a API do Bun
      const data = JSON.stringify(items); // Converte o array de objetos para uma string JSON
      return Bun.write(file, data); // Escreve os dados no arquivo de forma assíncrona
    } catch (error) {
      console.error('Error saving to file:', error); // Loga erro caso a escrita falhe
    }
  }

  // Método privado para carregar os dados do arquivo JSON
  private async loadFromFile() {
    const file = Bun.file(this.filepath); // Referencia o arquivo
    if (!(await file.exists())) // Verifica se o arquivo já existe no disco
      return [] // Se não existir, retorna um array vazio
    const data = await file.text(); // Lê o conteúdo do arquivo como texto
    // Converte o JSON de volta para instâncias da classe Item
    return JSON.parse(data).map((itemData: any) => new Item(itemData.description));
  }

  // Adiciona um novo item à lista e salva no arquivo
  async addItem(item: Item) {
    const items = await this.items; // Aguarda a lista atual
    items.push(item); // Adiciona o novo objeto Item ao array
    this.saveToFile(); // Dispara o salvamento (sem esperar o fim da escrita)
  }

  // Retorna todos os itens da lista
  async getItems() {
    return await this.items // Garante que os dados foram carregados antes de retornar
  }

  // Atualiza um item em uma posição específica
  async updateItem(index: number, newItem: Item) {
    const items = await this.items; // Obtém a lista
    if (index < 0 || index >= items.length) // Valida se o índice existe (correção aplicada)
      throw new Error('Index out of bounds'); // Lança erro se o índice for inválido
    items[index] = newItem; // Substitui o item antigo pelo novo
    this.saveToFile(); // Salva as alterações
  }

  // Remove um item da lista pelo seu índice
  async removeItem(index: number) {
    const items = await this.items; // Obtém a lista
    if (index < 0 || index >= items.length) // Valida o índice
      throw new Error('Index out of bounds');
    items.splice(index, 1); // Remove 1 elemento na posição indicada
    this.saveToFile(); // Atualiza o arquivo JSON
  }
}
```
```
import { ToDo, Item } from './core.ts'; // Importa a lógica de negócios

const todo = new ToDo('src/data.temp.json'); // Instancia o gerenciador com um arquivo temporário

// Configura o servidor usando o Bun.serve
const server = Bun.serve({
  port: 3001, // Define a porta de escuta do servidor
  async fetch(req) { // Função principal que lida com todas as requisições
    const url = new URL(req.url); // Faz o parse da URL recebida
    const path = url.pathname; // Extrai o caminho (ex: /items)
    const method = req.method; // Extrai o método HTTP (GET, POST, etc.)

    // Configuração de cabeçalhos para permitir requisições de outros domínios (CORS)
    const headers = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    // Responde prontamente a requisições de pré-verificação do navegador (Preflight)
    if (method === 'OPTIONS') {
      return new Response(null, { headers });
    }

    // Rota GET /items: Retorna a lista de tarefas
    if (path === '/items' && method === 'GET') {
      const items = await todo.getItems(); // Busca os itens no core
      return new Response(JSON.stringify(items), { headers }); // Retorna como JSON
    }

    // Rota POST /items: Adiciona uma nova tarefa
    if (path === '/items' && method === 'POST') {
      const body = await req.json(); // Extrai o corpo da requisição
      const item = new Item(body.description); // Cria novo objeto Item
      await todo.addItem(item); // Adiciona à lista
      return new Response(JSON.stringify(item), { status: 201, headers }); // Retorna sucesso
    }

    // Rotas que utilizam índice (UPDATE e DELETE)
    if (path.startsWith('/items/')) {
      const index = parseInt(path.split('/')[2]); // Extrai o número do índice da URL

      // Rota PUT /items/:index: Atualiza uma tarefa
      if (method === 'PUT') {
        const body = await req.json();
        const item = new Item(body.description);
        await todo.updateItem(index, item);
        return new Response(JSON.stringify(item), { headers });
      }

      // Rota DELETE /items/:index: Remove uma tarefa
      if (method === 'DELETE') {
        await todo.removeItem(index);
        return new Response(null, { status: 204, headers });
      }
    }

    // Servidor de arquivos estáticos (HTML/CSS do frontend)
    let filePath = path === '/' ? '/index.html' : path; // Define index.html como padrão
    const file = Bun.file(`public${filePath}`); // Busca o arquivo na pasta public
    if (await file.exists()) { // Se o arquivo existir no disco
      return new Response(file); // Retorna o conteúdo do arquivo (ex: a página web)
    }

    // Caso nenhuma rota coincida, retorna Erro 404
    return new Response('Not Found', { status: 404 });
  },
});

console.log(`Servidor rodando em http://localhost:${server.port}`); // Feedback no terminal
```
