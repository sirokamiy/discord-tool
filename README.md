失敗作）
import { pgTable, text, integer, boolean } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";

// User table schema
export const users = pgTable("users", {
  id: integer("id").primaryKey(),
  username: text("username").notNull(),
  password: text("password").notNull()
});

export const insertUserSchema = createInsertSchema(users).pick({
  username: true,
  password: true
});

// Channel table schema
export const channels = pgTable("channels", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  serverId: text("server_id").notNull(),
  serverName: text("server_name").notNull()
});

export const insertChannelSchema = createInsertSchema(channels);

// Message table schema
export const messages = pgTable("messages", {
  id: integer("id").primaryKey(),
  channelId: text("channel_id").notNull(),
  channelName: text("channel_name").notNull(),
  content: text("content").notNull(),
  sentAt: text("sent_at").notNull(),
  success: boolean("success").notNull(),
  errorMessage: text("error_message")
});

export const insertMessageSchema = createInsertSchema(messages).omit({
  id: true
});

// Discord configuration schema
export const discordConfig = z.object({
  tokens: z.string().min(1, "At least one Discord token is required")
});

// Channel selection schema
export const channelSelectSchema = z.object({
  channelId: z.string().min(1, "Channel ID is required"),
});

// Message sending schema
export const sendMessageSchema = z.object({
  channelId: z.string().min(1, "Channel ID is required"),
  content: z.string().min(1, "Message content is required").max(2000, "Message content cannot exceed 2000 characters"),
  repeatCount: z.number().int().min(1, "Repeat count must be at least 1").max(9999, "Repeat count cannot exceed 9999")
});

// Thread creation schema
export const createThreadSchema = z.object({
  channelId: z.string().min(1, "Channel ID is required"),
  threadName: z.string().min(1, "Thread name is required").max(100, "Thread name cannot exceed 100 characters"),
  initialMessage: z.string().min(1, "Initial message is required").max(2000, "Initial message cannot exceed 2000 characters"),
  repeatCount: z.number().int().min(1, "Repeat count must be at least 1").max(9999, "Repeat count cannot exceed 9999")
});

export type InsertUser = z.infer<typeof insertUserSchema>;
export type User = typeof users.$inferSelect;

export type InsertChannel = z.infer<typeof insertChannelSchema>;
export type Channel = typeof channels.$inferSelect;

export type InsertMessage = z.infer<typeof insertMessageSchema>;
export type Message = typeof messages.$inferSelect;

export type DiscordConfig = z.infer<typeof discordConfig>;
export type ChannelSelect = z.infer<typeof channelSelectSchema>;
export type SendMessage = z.infer<typeof sendMessageSchema>;
export type CreateThread = z.infer<typeof createThreadSchema>;
import { 
  User, InsertUser, 
  Channel, InsertChannel, 
  Message, InsertMessage
} from "@shared/schema";

export interface IStorage {
  getUser(id: number): Promise<User | undefined>;
  getUserByUsername(username: string): Promise<User | undefined>;
  createUser(user: InsertUser): Promise<User>;
  
  getChannels(): Promise<Channel[]>;
  getChannel(id: string): Promise<Channel | undefined>;
  createChannel(channel: InsertChannel): Promise<Channel>;
  
  getMessages(): Promise<Message[]>;
  getMessage(id: number): Promise<Message | undefined>;
  createMessage(message: InsertMessage): Promise<Message>;
}

export class MemStorage implements IStorage {
  private users: Map<number, User>;
  private channels: Map<string, Channel>;
  private messages: Map<number, Message>;
  currentUserId: number;
  currentMessageId: number;
  
  constructor() {
    this.users = new Map();
    this.channels = new Map();
    this.messages = new Map();
    this.currentUserId = 1;
    this.currentMessageId = 1;
  }
  
  async getUser(id: number): Promise<User | undefined> {
    return this.users.get(id);
  }
  
  async getUserByUsername(username: string): Promise<User | undefined> {
    return Array.from(this.users.values()).find(u => u.username === username);
  }
  
  async createUser(insertUser: InsertUser): Promise<User> {
    const id = this.currentUserId++;
    const user: User = { ...insertUser, id };
    this.users.set(id, user);
    return user;
  }
  
  async getChannels(): Promise<Channel[]> {
    return Array.from(this.channels.values());
  }
  
  async getChannel(id: string): Promise<Channel | undefined> {
    return this.channels.get(id);
  }
  
  async createChannel(channel: InsertChannel): Promise<Channel> {
    this.channels.set(channel.id, channel);
    return channel;
  }
  
  async getMessages(): Promise<Message[]> {
    return Array.from(this.messages.values());
  }
  
  async getMessage(id: number): Promise<Message | undefined> {
    return this.messages.get(id);
  }
  
  async createMessage(message: InsertMessage): Promise<Message> {
    const id = this.currentMessageId++;
    // Create a new message object with all required properties explicitly set
    const newMessage: Message = {
      id,
      channelId: message.channelId,
      content: message.content,
      channelName: message.channelName,
      sentAt: message.sentAt,
      success: message.success,
      errorMessage: message.errorMessage ?? null // Use nullish coalescing to ensure it's never undefined
    };
    this.messages.set(id, newMessage);
    return newMessage;
  }
}

export const storage = new MemStorage();
import type { Express } from "express";
import { createServer, type Server } from "http";
import { storage } from "./storage";
import { 
  discordConfig, 
  sendMessageSchema, 
  insertChannelSchema, 
  insertMessageSchema,
  createThreadSchema
} from "@shared/schema";
import { ZodError } from "zod";
import { fromZodError } from "zod-validation-error";
import { Client, GatewayIntentBits, TextChannel } from "discord.js";

// 複数のDiscordクライアントを管理するための変数
let discordClients: Map<string, Client> = new Map();

export async function registerRoutes(app: Express): Promise<Server> {
  // Discord API routes
  
  // Connect to Discord API
  app.post("/api/discord/connect", async (req, res) => {
    try {
      const { tokens } = discordConfig.parse(req.body);
      
      // 既存のクライアントを全て切断
      for (const [_, client] of discordClients) {
        client.destroy();
      }
      discordClients.clear();
      
      // トークンを空白で分割
      const tokenList = tokens.split(' ').filter(token => token.trim() !== '');
      const connectionResults = [];
      let allSuccess = true;
      
      // 各トークンに対して接続を試みる
      for (const token of tokenList) {
        try {
          // 新規Discordクライアントを作成
          const client = new Client({ 
            intents: [
              GatewayIntentBits.Guilds,
              GatewayIntentBits.GuildMessages
            ] 
          });
          
          // Discord APIに接続
          await client.login(token);
          
          // 成功したクライアントを保存
          discordClients.set(token, client);
          
          connectionResults.push({
            token: token.substring(0, 5) + '...' + token.substring(token.length - 5),
            success: true,
            username: client.user?.username,
            id: client.user?.id
          });
        } catch (error: any) {
          allSuccess = false;
          connectionResults.push({
            token: token.substring(0, 5) + '...' + token.substring(token.length - 5),
            success: false,
            error: error.message
          });
        }
      }
      
      if (discordClients.size > 0) {
        res.status(200).json({ 
          success: true, 
          message: `Connected to Discord API with ${discordClients.size} account(s)`,
          connectionResults
        });
      } else {
        res.status(401).json({ 
          success: false, 
          message: "Failed to authenticate with any Discord account", 
          connectionResults
        });
      }
    } catch (error) {
      if (error instanceof ZodError) {
        const validationError = fromZodError(error);
        return res.status(400).json({ 
          success: false, 
          message: validationError.message 
        });
      }
      
      res.status(500).json({ 
        success: false, 
        message: "Internal server error" 
      });
    }
  });
  
  // Disconnect from Discord API
  app.post("/api/discord/disconnect", (req, res) => {
    if (discordClients.size > 0) {
      for (const [_, client] of discordClients) {
        client.destroy();
      }
      discordClients.clear();
      res.status(200).json({ 
        success: true, 
        message: "Disconnected from Discord API" 
      });
    } else {
      res.status(200).json({ 
        success: true, 
        message: "Already disconnected" 
      });
    }
  });
  
  // Get connection status
  app.get("/api/discord/status", (req, res) => {
    if (discordClients.size > 0) {
      const users = [];
      
      for (const [token, client] of discordClients) {
        if (client.user) {
          users.push({
            username: client.user.username,
            id: client.user.id,
            token: token.substring(0, 5) + '...' + token.substring(token.length - 5)
          });
        }
      }
      
      res.status(200).json({ 
        connected: true, 
        connectedCount: discordClients.size,
        users
      });
    } else {
      res.status(200).json({ connected: false });
    }
  });
  
  // Get available channels
  app.get("/api/discord/channels", async (req, res) => {
    if (discordClients.size === 0) {
      return res.status(401).json({ 
        success: false, 
        message: "Not connected to Discord API" 
      });
    }
    
    try {
      // 最初のクライアントからチャンネル一覧を取得
      const [_, firstClient] = Array.from(discordClients.entries())[0];
      const guilds = firstClient.guilds.cache;
      const channels: any[] = [];
      
      // Collect channels from each guild
      guilds.forEach(guild => {
        guild.channels.cache
          .filter(channel => channel.type === 0) // TextChannel
          .forEach(channel => {
            channels.push({
              id: channel.id,
              name: channel.name,
              serverId: guild.id,
              serverName: guild.name
            });
            
            // Add to storage
            storage.createChannel({
              id: channel.id,
              name: channel.name,
              serverId: guild.id,
              serverName: guild.name
            });
          });
      });
      
      res.status(200).json({ channels });
    } catch (error: any) {
      res.status(500).json({ 
        success: false, 
        message: "Failed to fetch channels", 
        error: error.message 
      });
    }
  });
  
  // Send message to a channel
  app.post("/api/discord/send", async (req, res) => {
    if (discordClients.size === 0) {
      return res.status(401).json({ 
        success: false, 
        message: "Not connected to Discord API" 
      });
    }
    
    try {
      const { channelId, content, repeatCount } = sendMessageSchema.parse(req.body);
      
      // Get channel information
      const channelInfo = await storage.getChannel(channelId);
      let channelName = "unknown";
      
      if (channelInfo) {
        channelName = channelInfo.name;
      }
      
      const results = [];
      let overallSuccess = true;
      
      // 各クライアントでメッセージを送信
      for (const [token, client] of discordClients) {
        try {
          // チャンネルを取得
          const channel = await client.channels.fetch(channelId) as TextChannel;
          
          if (!channel || !channel.send) {
            throw new Error("Channel not found or cannot send messages to this channel");
          }
          
          // 指定回数メッセージを送信
          for (let i = 0; i < repeatCount; i++) {
            await channel.send(content);
            
            // 送信間隔を少し空ける（レート制限回避）
            if (i < repeatCount - 1) {
              await new Promise(resolve => setTimeout(resolve, 200));
            }
          }
          
          // 成功記録
          results.push({
            username: client.user?.username,
            success: true,
            messageCount: repeatCount
          });
          
          // 成功メッセージをストレージに記録
          await storage.createMessage({
            content,
            channelId,
            channelName,
            sentAt: new Date().toISOString(),
            success: true,
            errorMessage: null
          });
        } catch (error: any) {
          overallSuccess = false;
          
          // 失敗記録
          results.push({
            username: client.user?.username,
            success: false,
            error: error.message
          });
          
          // 失敗メッセージをストレージに記録
          await storage.createMessage({
            content,
            channelId,
            channelName,
            sentAt: new Date().toISOString(),
            success: false,
            errorMessage: error.message
          });
        }
      }
      
      if (overallSuccess) {
        res.status(200).json({ 
          success: true, 
          message: `Message sent successfully ${repeatCount} times using ${discordClients.size} account(s)`,
          channelName,
          results
        });
      } else {
        res.status(500).json({ 
          success: false, 
          message: "Some messages failed to send", 
          results
        });
      }
    } catch (error) {
      if (error instanceof ZodError) {
        const validationError = fromZodError(error);
        return res.status(400).json({ 
          success: false, 
          message: validationError.message 
        });
      }
      
      res.status(500).json({ 
        success: false, 
        message: "Internal server error" 
      });
    }
  });

  // Create threads in a channel
  app.post("/api/discord/create-thread", async (req, res) => {
    if (discordClients.size === 0) {
      return res.status(401).json({ 
        success: false, 
        message: "Not connected to Discord API" 
      });
    }
    
    try {
      const { channelId, threadName, initialMessage, repeatCount } = createThreadSchema.parse(req.body);
      
      // Get channel information
      const channelInfo = await storage.getChannel(channelId);
      let channelName = "unknown";
      
      if (channelInfo) {
        channelName = channelInfo.name;
      }
      
      const results = [];
      let overallSuccess = true;
      
      // 最初のクライアントのみでスレッドを作成（複数アカウントでの作成は制限あり）
      const [_, firstClient] = Array.from(discordClients.entries())[0];
      
      try {
        // Get the channel
        const channel = await firstClient.channels.fetch(channelId) as TextChannel;
        
        if (!channel || !channel.send) {
          throw new Error("Channel not found or cannot send messages to this channel");
        }
        
        // Create specified number of threads
        const createdThreads = [];
        
        for (let i = 0; i < repeatCount; i++) {
          // Create thread name with counter if creating multiple threads
          const currentThreadName = repeatCount > 1 ? `${threadName} (${i + 1}/${repeatCount})` : threadName;
          
          // Send initial message and create thread from it
          const message = await channel.send(initialMessage);
          const thread = await message.startThread({
            name: currentThreadName,
            autoArchiveDuration: 60, // Auto archive after 1 hour (60 minutes)
          });
          
          createdThreads.push({
            threadId: thread.id,
            threadName: currentThreadName
          });
          
          // Record successful message in storage
          await storage.createMessage({
            content: `Created thread: ${currentThreadName}`,
            channelId,
            channelName,
            sentAt: new Date().toISOString(),
            success: true,
            errorMessage: null
          });
          
          // スレッド作成間隔を少し空ける（レート制限回避）
          if (i < repeatCount - 1) {
            await new Promise(resolve => setTimeout(resolve, 500));
          }
        }
        
        results.push({
          username: firstClient.user?.username,
          success: true,
          threadCount: createdThreads.length,
          threads: createdThreads
        });
        
        res.status(200).json({ 
          success: true, 
          message: `Successfully created ${repeatCount} thread(s)`,
          channelName,
          createdThreads,
          results
        });
      } catch (error: any) {
        // Record failed message in storage
        const message = await storage.createMessage({
          content: `Failed to create thread: ${threadName}`,
          channelId,
          channelName,
          sentAt: new Date().toISOString(),
          success: false,
          errorMessage: error.message
        });
        
        res.status(500).json({ 
          success: false, 
          message: "Failed to create thread", 
          error: error.message 
        });
      }
    } catch (error) {
      if (error instanceof ZodError) {
        const validationError = fromZodError(error);
        return res.status(400).json({ 
          success: false, 
          message: validationError.message 
        });
      }
      
      res.status(500).json({ 
        success: false, 
        message: "Internal server error" 
      });
    }
  });

  const httpServer = createServer(app);
  return httpServer;
}
import { apiRequest } from "./queryClient";
import { SendMessage, DiscordConfig, Channel, CreateThread } from "@shared/schema";

// Connect to Discord API
export async function connectToDiscord(tokens: string) {
  return apiRequest("POST", "/api/discord/connect", { tokens });
}

// Disconnect from Discord API
export async function disconnectFromDiscord() {
  return apiRequest("POST", "/api/discord/disconnect", {});
}

// Get Discord connection status
export async function getDiscordStatus() {
  return apiRequest("GET", "/api/discord/status");
}

// Get available channels
export async function getDiscordChannels() {
  return apiRequest("GET", "/api/discord/channels");
}

// Send message to a channel
export async function sendDiscordMessage(message: SendMessage) {
  return apiRequest("POST", "/api/discord/send", message);
}

// Format text as bold
export function formatBold(text: string): string {
  return `**${text}**`;
}

// Format text as italic
export function formatItalic(text: string): string {
  return `*${text}*`;
}

// Format text as code block
export function formatCode(text: string): string {
  return `\`\`\`\n${text}\n\`\`\``;
}

// Format text as strikethrough
export function formatStrikethrough(text: string): string {
  return `~~${text}~~`;
}

// Create thread in a channel
export async function createDiscordThread(threadData: CreateThread) {
  return apiRequest("POST", "/api/discord/create-thread", threadData);
}
import { useState } from "react";
import { useMutation } from "@tanstack/react-query";
import { apiRequest } from "@/lib/queryClient";
import { discordConfig } from "@shared/schema";
import { zodResolver } from "@hookform/resolvers/zod";
import { useForm } from "react-hook-form";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { Eye, EyeOff, Loader2 } from "lucide-react";
import { StatusType } from "@/pages/Home";

interface ConnectionPanelProps {
  onConnectionChange: (connected: boolean) => void;
  showNotification: (type: StatusType, title: string, message: string) => void;
  refetchStatus: () => void;
}

export default function ConnectionPanel({ 
  onConnectionChange, 
  showNotification,
  refetchStatus
}: ConnectionPanelProps) {
  const [isTokenVisible, setIsTokenVisible] = useState(false);

  // Form setup
  const form = useForm({
    resolver: zodResolver(discordConfig),
    defaultValues: {
      tokens: ""
    }
  });

  // Connect mutation
  const connectMutation = useMutation({
    mutationFn: async (tokens: string) => {
      const res = await apiRequest("POST", "/api/discord/connect", { tokens });
      return res.json();
    },
    onSuccess: (data) => {
      if (data.success) {
        onConnectionChange(true);
        showNotification("success", "Connection Successful", 
          `Successfully connected to Discord API with ${data.connectionResults.filter(r => r.success).length} account(s)`);
        refetchStatus();
      } else {
        showNotification("error", "Connection Error", data.message || "Failed to connect to Discord");
      }
    },
    onError: (error: any) => {
      showNotification("error", "Connection Error", error.message || "Failed to connect to Discord");
    }
  });

  // Toggle token visibility
  const toggleTokenVisibility = () => {
    setIsTokenVisible(!isTokenVisible);
  };

  // Handle form submission
  const onSubmit = (data: { tokens: string }) => {
    connectMutation.mutate(data.tokens);
  };

  return (
    <div className="mb-8 bg-discord-dark rounded-md p-6 shadow-lg">
      <h2 className="text-lg font-medium mb-4">API Connection</h2>
      <Form {...form}>
        <form onSubmit={form.handleSubmit(onSubmit)} className="flex flex-col md:flex-row gap-4">
          <div className="flex-1">
            <FormField
              control={form.control}
              name="tokens"
              render={({ field }) => (
                <FormItem>
                  <FormLabel className="text-discord-light text-sm">Your Discord Token(s)</FormLabel>
                  <FormControl>
                    <div className="relative">
                      <Input
                        type={isTokenVisible ? "text" : "password"}
                        placeholder="Enter Discord token(s) separated by spaces"
                        className="w-full bg-discord-darker border border-gray-700 rounded-md px-4 py-2 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary"
                        {...field}
                      />
                      <button
                        type="button"
                        onClick={toggleTokenVisibility}
                        className="absolute right-2 top-1/2 transform -translate-y-1/2 text-discord-light hover:text-discord-lighter"
                      >
                        {isTokenVisible ? (
                          <EyeOff className="h-4 w-4" />
                        ) : (
                          <Eye className="h-4 w-4" />
                        )}
                      </button>
                    </div>
                  </FormControl>
                  <p className="text-xs text-discord-light mt-1">
                    Multiple tokens can be entered separated by spaces. All tokens are stored locally only.
                  </p>
                  <FormMessage />
                </FormItem>
              )}
            />
          </div>
          <div className="flex items-end">
            <Button
              type="submit"
              className="bg-discord-primary hover:bg-discord-primary-hover text-white font-medium px-6 py-2 rounded-md transition-colors h-10"
              disabled={connectMutation.isPending}
            >
              {connectMutation.isPending ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Connecting...
                </>
              ) : (
                "Connect"
              )}
            </Button>
          </div>
        </form>
      </Form>
    </div>
  );
}
import { useState, useRef } from "react";
import { useMutation } from "@tanstack/react-query";
import { apiRequest } from "@/lib/queryClient";
import { Channel, sendMessageSchema, createThreadSchema } from "@shared/schema";
import { zodResolver } from "@hookform/resolvers/zod";
import { useForm } from "react-hook-form";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { Textarea } from "@/components/ui/textarea";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs";
import { StatusType } from "@/pages/Home";
import { Bold, Italic, Code, Strikethrough, Send, Loader2, MessageSquarePlus } from "lucide-react";

interface MessagePanelProps {
  isConnected: boolean;
  channels: Channel[];
  showNotification: (type: StatusType, title: string, message: string) => void;
}

export default function MessagePanel({ 
  isConnected, 
  channels,
  showNotification 
}: MessagePanelProps) {
  const textareaRef = useRef<HTMLTextAreaElement>(null);
  const threadMessageRef = useRef<HTMLTextAreaElement>(null);
  const [characterCount, setCharacterCount] = useState(0);
  const [threadMessageCount, setThreadMessageCount] = useState(0);

  // Message form setup
  const form = useForm({
    resolver: zodResolver(sendMessageSchema),
    defaultValues: {
      channelId: "",
      content: "",
      repeatCount: 1
    }
  });

  // Thread form setup
  const threadForm = useForm({
    resolver: zodResolver(createThreadSchema),
    defaultValues: {
      channelId: "",
      threadName: "",
      initialMessage: "",
      repeatCount: 1
    }
  });

  // Send message mutation
  const sendMessageMutation = useMutation({
    mutationFn: async (data: { channelId: string; content: string; repeatCount: number }) => {
      const res = await apiRequest("POST", "/api/discord/send", data);
      return res.json();
    },
    onSuccess: (data) => {
      if (data.success) {
        form.setValue("content", "");
        setCharacterCount(0);
        showNotification(
          "success", 
          "Message Sent", 
          `Your message was successfully sent ${form.getValues("repeatCount")} time(s) to #${data.channelName}`
        );
      } else {
        showNotification("error", "Send Error", data.message || "Failed to send message");
      }
    },
    onError: (error: any) => {
      showNotification("error", "Send Error", error.message || "Failed to send message");
    }
  });

  // Create thread mutation
  const createThreadMutation = useMutation({
    mutationFn: async (data: { channelId: string; threadName: string; initialMessage: string; repeatCount: number }) => {
      const res = await apiRequest("POST", "/api/discord/create-thread", data);
      return res.json();
    },
    onSuccess: (data) => {
      if (data.success) {
        threadForm.setValue("threadName", "");
        threadForm.setValue("initialMessage", "");
        setThreadMessageCount(0);
        showNotification(
          "success", 
          "Thread Created", 
          `Successfully created ${data.createdThreads.length} thread(s) in #${data.channelName}`
        );
      } else {
        showNotification("error", "Thread Creation Error", data.message || "Failed to create thread");
      }
    },
    onError: (error: any) => {
      showNotification("error", "Thread Creation Error", error.message || "Failed to create thread");
    }
  });

  // Apply formatting to selected text
  const applyFormatting = (formatType: string) => {
    if (!textareaRef.current) return;
    
    const textarea = textareaRef.current;
    const start = textarea.selectionStart;
    const end = textarea.selectionEnd;
    const selectedText = textarea.value.substring(start, end);
    let formattedText = '';
    
    switch(formatType) {
      case 'bold':
        formattedText = `**${selectedText}**`;
        break;
      case 'italic':
        formattedText = `*${selectedText}*`;
        break;
      case 'code':
        formattedText = `\`\`\`\n${selectedText}\n\`\`\``;
        break;
      case 'strikethrough':
        formattedText = `~~${selectedText}~~`;
        break;
    }
    
    const newContent = 
      textarea.value.substring(0, start) + 
      formattedText + 
      textarea.value.substring(end);
    
    form.setValue("content", newContent, { shouldValidate: true });
    setCharacterCount(newContent.length);
    
    // Set focus back to textarea
    setTimeout(() => {
      textarea.focus();
      textarea.setSelectionRange(
        start + formattedText.length,
        start + formattedText.length
      );
    }, 0);
  };
  
  // Handle content change
  const handleContentChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    const content = e.target.value;
    setCharacterCount(content.length);
    form.setValue("content", content, { shouldValidate: true });
  };

  // Handle thread message content change
  const handleThreadMessageChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    const content = e.target.value;
    setThreadMessageCount(content.length);
    threadForm.setValue("initialMessage", content, { shouldValidate: true });
  };

  // Handle form submission
  const onSubmit = (data: { channelId: string; content: string; repeatCount: number }) => {
    sendMessageMutation.mutate(data);
  };

  // Handle thread form submission
  const onThreadSubmit = (data: { channelId: string; threadName: string; initialMessage: string; repeatCount: number }) => {
    createThreadMutation.mutate(data);
  };

  const getCharacterCountClass = () => {
    if (characterCount > 1900 && characterCount <= 2000) {
      return "text-discord-yellow";
    } else if (characterCount > 2000) {
      return "text-discord-red";
    }
    return "";
  };

  const getThreadMessageCountClass = () => {
    if (threadMessageCount > 1900 && threadMessageCount <= 2000) {
      return "text-discord-yellow";
    } else if (threadMessageCount > 2000) {
      return "text-discord-red";
    }
    return "";
  };

  return (
    <div 
      className={`bg-discord-dark rounded-md p-6 shadow-lg ${
        !isConnected ? "opacity-50 pointer-events-none" : ""
      } transition-opacity duration-200`}
    >
      <h2 className="text-lg font-medium mb-4">Discord Message Tool</h2>
      
      <Tabs defaultValue="message" className="w-full">
        <TabsList className="grid grid-cols-2 mb-4">
          <TabsTrigger value="message" className="data-[state=active]:bg-discord-primary">Send Message</TabsTrigger>
          <TabsTrigger value="thread" className="data-[state=active]:bg-discord-primary">Create Thread</TabsTrigger>
        </TabsList>
        
        <TabsContent value="message" className="mt-2">
          <Form {...form}>
            <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
              {/* Channel Selector */}
              <FormField
                control={form.control}
                name="channelId"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Channel</FormLabel>
                    <Select
                      disabled={!isConnected}
                      onValueChange={field.onChange}
                      defaultValue={field.value}
                    >
                      <FormControl>
                        <SelectTrigger className="bg-discord-darker border border-gray-700 rounded-md px-4 py-2 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary">
                          <SelectValue placeholder="Select a channel" />
                        </SelectTrigger>
                      </FormControl>
                      <SelectContent className="bg-discord-darker border border-gray-700 text-discord-lighter">
                        {channels.map((channel) => (
                          <SelectItem 
                            key={channel.id} 
                            value={channel.id}
                            className="focus:bg-discord-bg focus:text-discord-lighter hover:bg-discord-bg hover:text-discord-lighter"
                          >
                            #{channel.name} ({channel.serverName})
                          </SelectItem>
                        ))}
                      </SelectContent>
                    </Select>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Message Input */}
              <FormField
                control={form.control}
                name="content"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Message</FormLabel>
                    <div className="relative">
                      <FormControl>
                        <Textarea
                          {...field}
                          ref={textareaRef}
                          placeholder="Type your message here..."
                          disabled={!isConnected}
                          className="w-full bg-discord-darker border border-gray-700 rounded-md px-4 py-3 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary resize-none h-32"
                          onChange={handleContentChange}
                        />
                      </FormControl>
                      <div className="absolute right-2 bottom-2 flex gap-2 text-discord-light">
                        <Button
                          type="button"
                          variant="ghost"
                          size="icon"
                          disabled={!isConnected}
                          className="hover:text-discord-lighter p-1 rounded-md hover:bg-discord-bg transition-colors h-8 w-8"
                          onClick={() => applyFormatting('bold')}
                          title="Bold"
                        >
                          <Bold className="h-4 w-4" />
                        </Button>
                        <Button
                          type="button"
                          variant="ghost"
                          size="icon"
                          disabled={!isConnected}
                          className="hover:text-discord-lighter p-1 rounded-md hover:bg-discord-bg transition-colors h-8 w-8"
                          onClick={() => applyFormatting('italic')}
                          title="Italic"
                        >
                          <Italic className="h-4 w-4" />
                        </Button>
                        <Button
                          type="button"
                          variant="ghost"
                          size="icon"
                          disabled={!isConnected}
                          className="hover:text-discord-lighter p-1 rounded-md hover:bg-discord-bg transition-colors h-8 w-8"
                          onClick={() => applyFormatting('code')}
                          title="Code Block"
                        >
                          <Code className="h-4 w-4" />
                        </Button>
                        <Button
                          type="button"
                          variant="ghost"
                          size="icon"
                          disabled={!isConnected}
                          className="hover:text-discord-lighter p-1 rounded-md hover:bg-discord-bg transition-colors h-8 w-8"
                          onClick={() => applyFormatting('strikethrough')}
                          title="Strike-through"
                        >
                          <Strikethrough className="h-4 w-4" />
                        </Button>
                      </div>
                    </div>
                    <p className="text-xs text-discord-light mt-1">
                      <span className={getCharacterCountClass()}>
                        {characterCount}
                      </span>/2000 characters
                    </p>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Repeat Count */}
              <FormField
                control={form.control}
                name="repeatCount"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Repeat Count</FormLabel>
                    <FormControl>
                      <Input
                        type="number"
                        min={1}
                        max={9999}
                        {...field}
                        onChange={(e) => field.onChange(parseInt(e.target.value) || 1)}
                        placeholder="Number of times to send the message"
                        disabled={!isConnected}
                        className="bg-discord-darker border border-gray-700 rounded-md px-4 py-2 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary"
                      />
                    </FormControl>
                    <p className="text-xs text-discord-light mt-1">Number of times to send the message (1-9999)</p>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Send Button */}
              <div className="flex justify-end">
                <Button
                  type="submit"
                  className="bg-discord-primary hover:bg-discord-primary-hover text-white font-medium px-6 py-2 rounded-md transition-colors flex items-center"
                  disabled={
                    !isConnected || 
                    characterCount > 2000 || 
                    characterCount === 0 || 
                    !form.getValues("channelId") ||
                    sendMessageMutation.isPending
                  }
                >
                  {sendMessageMutation.isPending ? (
                    <>
                      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                      Sending...
                    </>
                  ) : (
                    <>
                      <Send className="mr-2 h-4 w-4" />
                      Send Message
                    </>
                  )}
                </Button>
              </div>
            </form>
          </Form>
        </TabsContent>
        
        <TabsContent value="thread" className="mt-2">
          <Form {...threadForm}>
            <form onSubmit={threadForm.handleSubmit(onThreadSubmit)} className="space-y-4">
              {/* Channel Selector */}
              <FormField
                control={threadForm.control}
                name="channelId"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Channel</FormLabel>
                    <Select
                      disabled={!isConnected}
                      onValueChange={field.onChange}
                      defaultValue={field.value}
                    >
                      <FormControl>
                        <SelectTrigger className="bg-discord-darker border border-gray-700 rounded-md px-4 py-2 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary">
                          <SelectValue placeholder="Select a channel" />
                        </SelectTrigger>
                      </FormControl>
                      <SelectContent className="bg-discord-darker border border-gray-700 text-discord-lighter">
                        {channels.map((channel) => (
                          <SelectItem 
                            key={channel.id} 
                            value={channel.id}
                            className="focus:bg-discord-bg focus:text-discord-lighter hover:bg-discord-bg hover:text-discord-lighter"
                          >
                            #{channel.name} ({channel.serverName})
                          </SelectItem>
                        ))}
                      </SelectContent>
                    </Select>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Thread Name */}
              <FormField
                control={threadForm.control}
                name="threadName"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Thread Name</FormLabel>
                    <FormControl>
                      <Input
                        {...field}
                        placeholder="Enter thread name..."
                        disabled={!isConnected}
                        className="bg-discord-darker border border-gray-700 rounded-md px-4 py-2 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary"
                      />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Initial Message */}
              <FormField
                control={threadForm.control}
                name="initialMessage"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Initial Message</FormLabel>
                    <div className="relative">
                      <FormControl>
                        <Textarea
                          {...field}
                          ref={threadMessageRef}
                          placeholder="Type the thread's initial message..."
                          disabled={!isConnected}
                          className="w-full bg-discord-darker border border-gray-700 rounded-md px-4 py-3 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary resize-none h-32"
                          onChange={handleThreadMessageChange}
                        />
                      </FormControl>
                    </div>
                    <p className="text-xs text-discord-light mt-1">
                      <span className={getThreadMessageCountClass()}>
                        {threadMessageCount}
                      </span>/2000 characters
                    </p>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Repeat Count */}
              <FormField
                control={threadForm.control}
                name="repeatCount"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel className="text-discord-light text-sm">Repeat Count</FormLabel>
                    <FormControl>
                      <Input
                        type="number"
                        min={1}
                        max={9999}
                        {...field}
                        onChange={(e) => field.onChange(parseInt(e.target.value) || 1)}
                        placeholder="Number of threads to create"
                        disabled={!isConnected}
                        className="bg-discord-darker border border-gray-700 rounded-md px-4 py-2 text-discord-lighter focus:outline-none focus:ring-2 focus:ring-discord-primary"
                      />
                    </FormControl>
                    <p className="text-xs text-discord-light mt-1">Number of identical threads to create (1-9999)</p>
                    <FormMessage />
                  </FormItem>
                )}
              />
              
              {/* Create Thread Button */}
              <div className="flex justify-end">
                <Button
                  type="submit"
                  className="bg-discord-primary hover:bg-discord-primary-hover text-white font-medium px-6 py-2 rounded-md transition-colors flex items-center"
                  disabled={
                    !isConnected || 
                    threadMessageCount > 2000 || 
                    threadMessageCount === 0 || 
                    !threadForm.getValues("channelId") ||
                    !threadForm.getValues("threadName") ||
                    createThreadMutation.isPending
                  }
                >
                  {createThreadMutation.isPending ? (
                    <>
                      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                      Creating...
                    </>
                  ) : (
                    <>
                      <MessageSquarePlus className="mr-2 h-4 w-4" />
                      Create Thread
                    </>
                  )}
                </Button>
              </div>
            </form>
          </Form>
        </TabsContent>
      </Tabs>
    </div>
  );
}
import { FaDiscord } from "react-icons/fa";

interface HeaderProps {
  isConnected: boolean;
}

export default function Header({ isConnected }: HeaderProps) {
  return (
    <header className="bg-discord-darker px-6 py-4 shadow-md">
      <div className="container mx-auto flex flex-col">
        <div className="text-xs text-discord-light text-center mb-1">
          made sirokami しろかみが作りました
        </div>
        <div className="flex items-center justify-between">
          <h1 className="text-xl font-bold flex items-center">
            <FaDiscord className="text-discord-primary mr-2" />
            Discord Message Sender
          </h1>
          <div className="text-discord-light text-sm">
            <span id="connection-status" className="flex items-center">
              {isConnected ? (
                <>
                  <i className="fas fa-circle text-discord-green mr-2"></i>
                  Connected
                </>
              ) : (
                <>
                  <i className="fas fa-circle text-discord-red mr-2"></i>
                  Disconnected
                </>
              )}
            </span>
          </div>
        </div>
      </div>
    </header>
  );
}
import { useState, useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import { Channel } from "@shared/schema";
import { getDiscordStatus, getDiscordChannels, disconnectFromDiscord } from "@/lib/discord";
import Header from "@/components/Header";
import ConnectionPanel from "@/components/ConnectionPanel";
import MessagePanel from "@/components/MessagePanel";
import Footer from "@/components/Footer";
import { ToastAction } from "@/components/ui/toast";
import { useToast } from "@/hooks/use-toast";

export type StatusType = "success" | "error" | "warning";

export interface StatusNotificationProps {
  type: StatusType;
  title: string;
  message: string;
  visible: boolean;
}

export default function Home() {
  const [isConnected, setIsConnected] = useState(false);
  const [channels, setChannels] = useState<Channel[]>([]);
  const { toast } = useToast();

  // Get Discord status
  const { data: statusData, refetch: refetchStatus } = useQuery({
    queryKey: ['discord-status'],
    queryFn: async () => {
      const response = await getDiscordStatus();
      return response.json();
    },
    refetchOnWindowFocus: false
  });

  // Get Discord channels
  const { data: channelsData, refetch: refetchChannels } = useQuery({
    queryKey: ['discord-channels'],
    queryFn: async () => {
      const response = await getDiscordChannels();
      return response.json();
    },
    refetchOnWindowFocus: false,
    enabled: isConnected
  });

  // Update connection status when status data changes
  useEffect(() => {
    if (statusData) {
      setIsConnected(statusData.connected || false);
      
      if (statusData.connected) {
        // 接続済みアカウント数を表示
        showNotification(
          "success", 
          "Connection Status", 
          `Connected with ${statusData.connectedCount || 1} account(s)`
        );
        refetchChannels();
      }
    }
  }, [statusData, refetchChannels]);

  // Update channels when channels data changes
  useEffect(() => {
    if (channelsData && channelsData.channels) {
      setChannels(channelsData.channels);
    }
  }, [channelsData]);

  // Handle connection state change
  const handleConnectionChange = (connected: boolean) => {
    setIsConnected(connected);
    
    if (connected) {
      refetchChannels();
    } else {
      setChannels([]);
    }
  };

  // Display notification toast
  const showNotification = (type: StatusType, title: string, message: string) => {
    toast({
      variant: type === "error" ? "destructive" : "default",
      title: title,
      description: message,
      action: type === "error" ? (
        <ToastAction altText="Retry">Retry</ToastAction>
      ) : undefined,
    });
  };

  return (
    <div className="min-h-screen bg-discord-bg text-white flex flex-col">
      <Header isConnected={isConnected} />
      
      <main className="container mx-auto px-4 py-8 flex-1">
        <ConnectionPanel 
          onConnectionChange={handleConnectionChange}
          showNotification={showNotification}
          refetchStatus={refetchStatus}
        />
        
        <MessagePanel 
          isConnected={isConnected}
          channels={channels}
          showNotification={showNotification}
        />
      </main>
      
      <Footer />
    </div>
  );
}
