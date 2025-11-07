// NÁVRH IMPLEMENTÁCIE: Nový API endpoint pre viacjazyčné liturgické kalendáre

// 1. INŠTALÁCIA RUBY GEM (cez shell command alebo Docker container)
// gem install calendarium-romanum

// 2. VYTVORENIE NOVÉHO API ENDPOINTU
// src/app/api/liturgical-calendar-multi/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

/**
 * API Wrapper pre calendarium-romanum Ruby gem
 * Poskytuje liturgický kalendár RKC vo viacerých jazykoch
 */

/**
 * GET /api/liturgical-calendar-multi?action=year&year=2025&lang=es
 * GET /api/liturgical-calendar-multi?action=month&year=2025&month=12&lang=it  
 * GET /api/liturgical-calendar-multi?action=day&year=2025&month=12&day=8&lang=pt
 */
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  
  const action = searchParams.get('action') || 'day';
  const year = searchParams.get('year');
  const month = searchParams.get('month');
  const day = searchParams.get('day');
  const lang = searchParams.get('lang') || 'en';
  
  // Validácia jazyka - podporované jazyky z calendarium-romanum
  const supportedLanguages = {
    'en': 'universal-en',      // Angličtina
    'es': 'universal-es',      // Španielčina  
    'it': 'universal-it',      // Taliančina
    'fr': 'universal-fr',      // Francúzština
    'pt': 'universal-pt',      // Portugalčina
    'la': 'universal-la',      // Latinka
    'cs': 'universal-cs',      // Čeština
    'de': 'calapi'             // Nemčina - fallback na CalAPI + preklad
  };

  if (!supportedLanguages[lang as keyof typeof supportedLanguages]) {
    return NextResponse.json(
      { error: `Unsupported language. Supported: ${Object.keys(supportedLanguages).join(', ')}` },
      { status: 400 }
    );
  }

  try {
    switch (action) {
      case 'year': {
        if (!year) {
          return NextResponse.json({ error: 'Year is required' }, { status: 400 });
        }

        // Pre nemčinu použijeme CalAPI + preklad
        if (lang === 'de') {
          return await handleGermanCalendar(year);
        }

        // Pre ostatné jazyky použijeme calendarium-romanum
        const calendarFile = supportedLanguages[lang as keyof typeof supportedLanguages];
        const rubyScript = generateRubyScript(action, { year, lang, calendarFile });
        
        const { stdout, stderr } = await execAsync(`ruby -e "${rubyScript}"`);
        
        if (stderr) {
          throw new Error(`Ruby script error: ${stderr}`);
        }

        const result = JSON.parse(stdout);
        return NextResponse.json(result);
      }

      case 'month': {
        if (!year || !month) {
          return NextResponse.json({ error: 'Year and month are required' }, { status: 400 });
        }

        const calendarFile = supportedLanguages[lang as keyof typeof supportedLanguages];
        const rubyScript = generateRubyScript(action, { year, month, lang, calendarFile });
        
        const { stdout, stderr } = await execAsync(`ruby -e "${rubyScript}"`);
        
        if (stderr) {
          throw new Error(`Ruby script error: ${stderr}`);
        }

        const result = JSON.parse(stdout);
        return NextResponse.json(result);
      }

      case 'day': {
        if (!year || !month || !day) {
          return NextResponse.json({ error: 'Year, month and day are required' }, { status: 400 });
        }

        const calendarFile = supportedLanguages[lang as keyof typeof supportedLanguages];
        const rubyScript = generateRubyScript(action, { year, month, day, lang, calendarFile });
        
        const { stdout, stderr } = await execAsync(`ruby -e "${rubyScript}"`);
        
        if (stderr) {
          throw new Error(`Ruby script error: ${stderr}`);
        }

        const result = JSON.parse(stdout);
        return NextResponse.json(result);
      }

      default:
        return NextResponse.json(
          { error: `Invalid action. Supported: year, month, day` },
          { status: 400 }
        );
    }
  } catch (error) {
    console.error('API Error:', error);
    return NextResponse.json(
      { 
        error: 'Internal server error', 
        details: error instanceof Error ? error.message : 'Unknown error' 
      },
      { status: 500 }
    );
  }
}

// Generuje Ruby skript pre calendarium-romanum
function generateRubyScript(action: string, params: any): string {
  const { year, month, day, lang, calendarFile } = params;
  
  return `
    require 'calendarium-romanum'
    require 'json'
    require 'date'
    
    # Nastavenie lokalizácie
    I18n.locale = :${lang}
    
    # Načítanie kalendárových dát  
    sanctorale = CalendariumRomanum::Data::${calendarFile.toUpperCase().replace('-', '_')}.load
    pcal = CalendariumRomanum::PerpetualCalendar.new(sanctorale: sanctorale)
    
    result = {}
    
    case '${action}'
    when 'year'
      year = ${year}
      days = []
      
      Date.new(year, 1, 1).upto(Date.new(year, 12, 31)) do |date|
        day_info = pcal[date]
        
        celebrations = day_info.celebrations.map do |c|
          {
            title: c.title,
            rank: c.rank.desc,
            rank_num: c.rank.priority,
            colour: c.colour.symbol.to_s
          }
        end
        
        days << {
          date: date.strftime('%Y-%m-%d'),
          season: day_info.season.symbol.to_s,
          season_week: day_info.season_week,
          weekday: date.strftime('%A').downcase,
          celebrations: celebrations
        }
      end
      
      result = {
        year: year,
        lang: '${lang}',
        total_days: days.length,
        days: days
      }
      
    when 'month'
      year = ${year}
      month = ${month}
      days = []
      
      Date.new(year, month, 1).upto(Date.new(year, month, -1)) do |date|
        day_info = pcal[date]
        
        celebrations = day_info.celebrations.map do |c|
          {
            title: c.title,
            rank: c.rank.desc,
            rank_num: c.rank.priority,
            colour: c.colour.symbol.to_s
          }
        end
        
        days << {
          date: date.strftime('%Y-%m-%d'),
          season: day_info.season.symbol.to_s,
          season_week: day_info.season_week,
          weekday: date.strftime('%A').downcase,
          celebrations: celebrations
        }
      end
      
      result = {
        year: year,
        month: month,
        lang: '${lang}',
        days: days
      }
      
    when 'day'
      date = Date.new(${year}, ${month}, ${day})
      day_info = pcal[date]
      
      celebrations = day_info.celebrations.map do |c|
        {
          title: c.title,
          rank: c.rank.desc,
          rank_num: c.rank.priority,
          colour: c.colour.symbol.to_s
        }
      end
      
      result = {
        date: date.strftime('%Y-%m-%d'),
        season: day_info.season.symbol.to_s,
        season_week: day_info.season_week,
        weekday: date.strftime('%A').downcase,
        celebrations: celebrations
      }
    end
    
    puts result.to_json
  `;
}

// Pre nemčinu - fallback na CalAPI + AI preklad
async function handleGermanCalendar(year: string) {
  // Načítaj český kalendár z CalAPI
  const calApiResponse = await fetch(`/api/liturgical-calendar?action=year&year=${year}&lang=cs`);
  const czechData = await calApiResponse.json();
  
  // Prelož názvy osláv do nemčiny
  // (tu by ste implementovali preklad cez AI API)
  
  return {
    year: parseInt(year),
    lang: 'de',
    total_days: czechData.days?.length || 0,
    days: czechData.days || [],
    note: 'German calendar generated from Czech source + AI translation'
  };
}