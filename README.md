/**
 * AST Clone Detector (ast_clone_detector.ts)
 *
 * Uses TypeScript compiler API to build normalized AST fingerprints and find similar functions.
 *
 * NOTE: install `npm install typescript` to run.
 *
 * Run: ts-node src/ast_clone_detector.ts <path_to_file.ts>
 */

import ts from 'typescript';
import fs from 'fs';
import path from 'path';
import crypto from 'crypto';

function fingerprintFunction(node: ts.FunctionLikeDeclarationBase, sourceFile: ts.SourceFile) {
  // collect token kinds sequence (normalized)
  const tokens: string[] = [];
  function walk(n: ts.Node) {
    tokens.push(ts.SyntaxKind[n.kind]);
    n.forEachChild(walk);
  }
  walk(node);
  // hash tokens
  return crypto.createHash('sha256').update(tokens.join(',')).digest('hex');
}

function analyzeFile(filePath: string) {
  const src = fs.readFileSync(filePath, 'utf8');
  const sf = ts.createSourceFile(filePath, src, ts.ScriptTarget.Latest, true);
  const map = new Map<string, { name?: string; pos: number }[]>();
  function visit(node: ts.Node) {
    if (ts.isFunctionDeclaration(node) || ts.isMethodDeclaration(node) || ts.isFunctionExpression(node)) {
      const fp = fingerprintFunction(node as any, sf);
      const name = (node as any).name ? String((node as any).name?.getText?.()) : '<anon>';
      map.set(fp, (map.get(fp) || []).concat({ name, pos: node.pos }));
    }
    ts.forEachChild(node, visit);
  }
  visit(sf);
  // find groups with >1
  for (const [k, v] of map.entries()) {
    if (v.length > 1) {
      console.log('Possible clones (fingerprint):', k);
      v.forEach(x => console.log(' -', x.name, 'pos', x.pos));
    }
  }
}

if (require.main === module) {
  const file = process.argv[2] || path.join(__dirname, '../sample.ts');
  if (!fs.existsSync(file)) { console.log('file not found', file); process.exit(1); }
  analyzeFile(file);
}
